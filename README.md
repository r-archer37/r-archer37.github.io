##Processing Streaming Data with Python

Think about code you've written to analyze data in python. If it's anything like data analysis code I've written, it started with loading data into memory from a file, or from a database. It's a very easy thing to do; python has very effective tools to load in data structured that way (as do R and any number of other programming languages).

Underlying the effectiveness of those tools is that storing many data points in one location in a row-and-column structure actually embeds a huge amount of information about the data: the number of observations, the grain of observations, and the available features of the observations are all implicit there. But what if we want to process data that arrives to us as it is generated -- that is to say, in a stream?

What I am going to demonstrate is Faust, a powerful tool for processing data streams using pure python. Faust is a python package (just `pip install faust`) that was made open-source by Robinhood: https://faust.readthedocs.io/en/latest/ The official documentation of the package is not especially lacking, but it can be hard to start using and understanding this kind of high-level stream-processing without an interactive demonstration. That's why part 2 of this article is an interactive demonstration that requires only a purpose-built github repo and Docker desktop to follow along with.

###Part 0: What is Streaming Data?

Let's start by being more concrete about what we mean by a stream. Within this article, I am going to talk about streams that look like a topic in Apache Kafka, but from the perspective of the code we'll be writing, it's not all that different from, say, the public Twitter API or any number of other streams out there. They all look like a queue, to which messages (typically JSON blobs) can be published, and from which messages can be consumed. The expected usage has a consumer taking one messsage at a time off the end of the queue, performing some operation on it, and then coming back for the next message.

For example a Kafka instance might have topics like `incoming_payment_received` and `incoming_payment_processed`. When our software application initially receives payment it might publish a message to the former, prompting the creation of a record in a payment database. Subsequently, when it has properly attributed that payment for accounting purposes, our application might publish a message to the latter topic, prompting an update to the corresponding record.

GRAPHIC GOES HERE?

A key feature of these streams is that a consumer can finish processing all messages in the queue, and be in a state of waiting for the next one to arrive, either up to some time-limit set by a developer, or indefinitely.

GRAPHIC GOES HERE?

###Part 1: Building a Faust App

For the purpose of this demonstration, let's assume we want to develop a process that will wait indefinitely for new messages, so long as the queue it is connected to exists. This is what Faust will call our `app`. The typical Faust app will have `agent`s, `topic`s, and connections between the them. Let's start by looking at a very simple app and then break down what's going on:

#### `my_example_faust_app.py`
```python
import faust

app = faust.App('example_faust_app',
                broker='kafka:9092',
                value_serializer='raw'
               )

input_stream = app.topic('example_kafka_topic')

@app.agent(input_stream)
async def report_messages_received(messages):
    async for message in messages:
        print(message)

if __name__ == '__main__':
    app.main()
```

Starting from the bottom of the code, the first block we see looks similar to how a lot of python code ends, by telling the program what to do when we simply execute this python file. The only difference is that we take advantage of Faust's App class' `main()` function instead of writing our own. This is great: we don't need to spend our time initializing logging or or sorting out the nitty-gritty of connecting to Kafka. Faust handles all of that for us.

```python
@app.agent(input_stream)
async def report_messages_received(messages):
    async for message in messages:
        print(message)
```

In this block, the bottom two lines do just what you probably think they do (we'll come to async in a moment), so let's instead focus on the top two lines which contain more exotic syntax. `async` tells us that this is a python loop or function that can be executed in parallel with other async python. The details of this are beyond the scope of this article (https://docs.python.org/3/library/asyncio.html), but our program can to listen to many topics at once, so processing messages in parallel is quite nice. The function name and input variable are arbitrary names. The decorator on the function `@app.agent(input_stream)` means that this function will be an instance of `faust.Agent` on the app topic we called `input_stream`. That is, messages coming from that stream will be routed to this topic, letting it iterate over an indefinite series of messages.

```python
input_stream = app.topic('example_kafka_topic')
```

This line defines our input stream, an instance of `faust.Topic`. By default, if the string is the name of a topic that exists in the Kafka server we connect to, our app will set up to consume from that topic; if the topic doesn't exist, our app will create it, and then set up to consume from it. Recall that consuming from a stream like this simply means taking one message at a time from a queue, doing whatever we've programmed our app to do, and then repeating.

```python
import faust

app = faust.App('example_faust_app',
                broker='kafka:9092',
                value_serializer='raw'
               )
```

This block defines the core of our program; an instance of `faust.App`. We give it a name, `'example_faust_app'`, tell it what message broker it will be using (`kafka:9092` is a default address for Kafka connections), and optionally declare a message serializer (e.g., 'raw', 'json', don't worry too much about this). We can also freely pass any other arguments here to instatiate our app with additional attributes.

###Aside: Running a Faust app

There's one last piece of code now standing between us and executing our fast application. If we simply enter `python my_example_faust_app.py` into our terminal, we'll get a usage guide, rather than actually executing our program (recall that we're executing `faust.App.main`). In order to actually start our program, we need to start a Faust worker -- a concept that isn't critical to understand to write a simple program, but can enable scaling and redundancy required for critical production systems. To do that simply provide the argument worker along with our termainl command: `python my_example_faust_app.py worker`. Now, so long as we're able to connect to Kafka at `kafka:9092`, we have a Faust app that will wait to process messages for as long as we let it.

###Part 2: Interactive Demonstration

For this demonstration, I make use of Jupyter Lab, Faust, Kafka, Zookeeper, and more. But in order to setup reasonable, everything is packged up within Docker containers. In order to follow along, all you'll need installed is a git client and Docker Desktop. First up, clone the repository: GITHUB REPO GOES HERE. Next, in a terminal, navigate to the root of the repository and run `docker-compose up`. This should generate a fair amount of text on your screen; in order to stop the process at any time you can use ctrl+c. If afterwards anything is lingering, you can stop it by executing `docker-compose down` in the same location.

Now, with the docker-compose still running, open a browser window and point it to `localhost:8888`. You should be prompted to enter a "Password or token" which is set to `faust`. Clicking Log In will bring you to the Jupyter Lab Launcher tab. From the Launcher, click Terminal. In the terminal, execute `python faust/simple_faust_demo.py worker`. If all goes well, you'll see a readout of app properties, a smiling face and a blinking cursor.

SCREENSHOT GOES HERE

Now, using the file browser on the left, open the `notebooks` folder, and then the `pykafka_producer.ipynb` notebook. We'll use this notebook to send messages to our Faust app via Kafka. It uses the `pykafka` package, rather than Faust, because this allows a much lower-level interface that is great for an interactive demo, but would be much tougher to build an application around. In that notebook, execute the first three cells:

#### pykafka_producer.ipynb
```python
# Cell 1 connects to Kafka
from pykafka import KafkaClient
import pykafka

from helpers.kafka_helpers import get_topic, produce_to_topic

KAFKA_ADDRESS = 'kafka:9092'

kafka_client = KafkaClient(KAFKA_ADDRESS)

assert all([broker.connected for broker in kafka_client.brokers.values()])
```
```python
# Cell 2 defines a topic
topic1 = get_topic(topic_name='example_kafka_topic1',  # creates a new topic if it doesn't already exist
                   client=kafka_client)
```
```python
# Cell 3 publishes a message to that topic
produce_to_topic(topic=topic1,  # topic must already exist
                 message='hello')
```

All of this is, of course, barely scratching the surface of what Faust can do. And of course a lot more is documented on the project's homepage: https://faust.readthedocs.io. However, it can be hard to really get a handle on this code if you don't already have a working kafka instance to play around with. If you really want to give it a try yourself, visit this GITHUB REPO I MADE where, using only Docker Desktop, you can set yourself up a nice interactive environment to play around with Kafka, Faust, and more!