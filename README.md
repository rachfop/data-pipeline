# Run data pipelines with Temporal

Temporal makes writing data pipelines easy with Workflows and Activities.

You can create a source, process the step or steps, and output the flow of information to a destination with just code. Meaning all of your developer best practices can be implemented, tested, and ran as needed.

That data that enters a Workflow is handled by Activities, while the Workflow orchestrates the execution of those steps.
You can ensure that Temporal handles all actions and executes it observably once, all in Python code.

In this tutorial, you'll learn to build a data pipeline that gets the top 10 Hacker New stories and processes the items based on the story ID.
If the API endpoint is down, the default behavior of the Retry Policy is to retry indefinitely.

## Step 0: Prerequisites

* Python >= 3.7
* [Poetry](https://python-poetry.org)
* [Local Temporal server running](https://docs.temporal.io/application-development/foundations#run-a-development-cluster)

With this repository cloned, run the following at the root of the directory:

```bash
poetry install
```

## Step 1: Workflow Definition

Write a Workflow Definition file that contains the steps that you want to execute.

```python
# your_workflow.py
from datetime import timedelta

from temporalio import workflow

with workflow.unsafe.imports_passed_through():
    from activities import hackernews_top_stories, hackernews_top_story_ids


@workflow.defn
class HackerNewsWorkflow:
    @workflow.run
    async def run(self) -> list:
        news_id = await workflow.execute_activity(
            hackernews_top_story_ids,
            start_to_close_timeout=timedelta(seconds=15),
        )
        return await workflow.execute_activity(
            hackernews_top_stories,
            news_id,
            start_to_close_timeout=timedelta(seconds=15),
        )
```

The `HackerNewsWorkflow` class is decorated with the `@workflow.defn` which must be set on any registered workflow class.

The `async def run()` function is decorated with the `@workflow.run` which is set on the one asynchronous method on the same class as the `@workflow.defn`.

There are two Activities being executed, `hackernews_top_story_ids` and `hackernews_top_stories`.
These Activities are defined in the `activities.py` file, which will be explained later.

Inside the `workflow.execute_activity()` function, pass the reference of Activity Definition, function, or step in your data pipeline.
If that step takes an argument, then use the second positional argument for that name, as shown in the second `execute_activity()` function.

You must set either a Start-To-Close or Schedule-To-Close Activity Timeout.

## Step 2: Activities

In the `activities.py` file, write out each step in the data processing pipeline.

For example, `hackernews_top_story_ids()` gets the top 10 stories from Hacker News while, `hackernews_top_stories()` gets items based on the stories ID.

Use the `aiohttp` library instead of `requests` to avoid making blocking calls.

```python
# activities.py
from dataclasses import dataclass

import aiohttp
from temporalio import activity

TASK_QUEUE_NAME = "hackernews-task-queue"


async def fetch(session, url):
    async with session.get(url) as response:
        return await response.json()


@activity.defn
async def hackernews_top_story_ids() -> list[int]:
    async with aiohttp.ClientSession() as session:
        async with session.get(
            "https://hacker-news.firebaseio.com/v0/topstories.json"
        ) as response:
            top_story_ids = await response.json()
    return top_story_ids[:10]


@activity.defn
async def hackernews_top_stories(hackernews_top_story_ids) -> list[list[str]]:
    results = []
    async with aiohttp.ClientSession() as session:
        for item_id in hackernews_top_story_ids:
            try:
                item = await fetch(
                    session,
                    f"https://hacker-news.firebaseio.com/v0/item/{item_id}.json",
                )
                results.append([item["title"], item["by"], item["url"]])
            except KeyError:
                print(f"Error processing item {item_id}: {item}")
    return results
```

Each function contains an `activity.defn` decorator that ensures that function is durably backed by Temporal.
The Retry Policy defined in the `HackerNewsWorkflow` class contains information needed to retry in case the API endpoint is down.

By default, the Retry Policy is:

```output
Initial Interval     = 1 second
Backoff Coefficient  = 2.0
Maximum Interval     = 100 × Initial Interval
Maximum Attempts     = ∞
Non-Retryable Errors = []
```

The last step of the data pipeline returns the results.

## Step 3: Run Worker

In the `run_worker.py` file, set the Worker to host the Workflows and/or Activities.

```python
# run_worker.py
import asyncio

from temporalio.client import Client
from temporalio.worker import Worker

from activities import TASK_QUEUE_NAME, hackernews_top_stories, hackernews_top_story_ids
from your_workflow import HackerNewsWorkflow


async def main():
    client = await Client.connect("localhost:7233")
    worker = Worker(
        client,
        task_queue=TASK_QUEUE_NAME,
        workflows=[HackerNewsWorkflow],
        activities=[hackernews_top_stories, hackernews_top_story_ids],
    )
    await worker.run()


if __name__ == "__main__":
    asyncio.run(main())
```

This Worker run creates and uses the same Client used for starting the Workflow, `localhost:7233`.
The Worker must be set to the same Task Queue name, then specify your Workflows and Activities names in a list.

Then run the Worker with the [asyncio.run()](https://docs.python.org/3/library/asyncio-runner.html#asyncio.run) function.

## Step 4: Run your Workflow

The file `run_workflow.py` processes the Execution of the Workflow.
To start, you connect to an instance of the Temporal Client. Since it is running locally, it is connected to `localhost:7233`.

Then it executes the Workflow, by passing the name of the Workflow, a Workflow ID, and a Task Queue name.

```python
# run_workflow.py
import asyncio

import pandas as pd
from temporalio.client import Client

from activities import TASK_QUEUE_NAME
from your_workflow import HackerNewsWorkflow


async def main():
    client = await Client.connect("localhost:7233")

    data = await client.execute_workflow(
        HackerNewsWorkflow.run,
        id="hackernews-workflow",
        task_queue=TASK_QUEUE_NAME,
    )

    df = pd.DataFrame(data)
    print(df)
    return df


if __name__ == "__main__":
    asyncio.run(main())
```

When the Workflow process it steps, it will finally return the `data` variable. For this example, `data` is processed by a Pandas Data frame.

The code is run in the `asyncio` event loop.

## Step 5: Results

To run your code, open two terminal windows and run the following:

```bash
# terminal one
poetry run python run_worker.py
# terminal two
poetry run python run_workflow.py
```

You should see something similar to the following in your terminal.

```output
                                                   0  ...                                                  2
0  Could we stop Yellowstone from erupting with a...  ...  https://constructionphysics.substack.com/p/cou...
1  Sandy Munro Talks Battery Battles, Calls Solid...  ...  https://www.sae.org/blog/sandy-munro-live-sae-wcx
2  Microsoft Edge is leaking the sites you visit ...  ...  https://www.theverge.com/2023/4/25/23697532/mi...
3  Commercial lunar lander presumed lost after mo...  ...  https://www.cnn.com/2023/04/25/world/lunar-lan...
4                Fun with Kermit and ZMODEM over SSH  ...  https://www.cambus.net/fun-with-kermit-and-zmo...
5  Smartphones with Qualcomm chip secretly send p...  ...  https://www.nitrokey.com/news/2023/smartphones...
6       A non-technical explanation of deep learning  ...  https://www.parand.com/a-completely-non-techni...
7    Use Gröbner bases to solve polynomial equations  ...    https://jingnanshi.com/blog/groebner_basis.html
8                        Eww: ElKowars wacky widgets  ...                     https://github.com/elkowar/eww
9  FSF Call on the IRS to provide libre tax-filin...  ...  https://www.fsf.org/blogs/community/call-on-th...
[10 rows x 3 columns]
```

Now go to your running instance of the [Temporal Web UI](http://localhost:8233/namespaces/default/workflows), to see how the information is persisted in history.

1. Select the most recently running Workflow by Workflow ID, for example `hackernews-workflow`.
2. Open the **Input and results** pane to see what was entered and returned to the Workflow.
3. Under **Recent Events,** you can observe every step and task created by the data pipeline.
    This information is persisted in History, meaning that if any point a failure is created in your data pipeline, you can resume from that point in the history, rather than starting over from the beginning.

## Conclusion

You have learned how to create and process data with a data pipeline that's durably backed by Temporal.

With Temporal, you have insight into your data pipelines. You can see every point in History and have the ability to resume from a failure or retry.

### Next steps

Now on your own, write another Activity, or step in your data pipeline, that extracts the most frequently occurring words or topics in the story title.

* How do you tell the Worker to process that information?
* How does the Workflow know to process that step?
* What is returned by the Workflow Execution?
