---
title: Storm and Kestrel
layout: documentation
documentation: true
---
本页介绍如何使用Storm消费来自Kestrel集群的项目。

## 准备阶段
### Storm
本教程使用的示例来自于 [storm-kestrel](https://github.com/nathanmarz/storm-kestrel) 项目和 [storm-starter](http://github.com/apache/storm/blob/{{page.version}}/examples/storm-starter) 项目。建议你克隆这些项目并根着示例走。 阅读 [Setting up development environment](Setting-up-development-environment.html) 和 [Creating a new Storm project](Creating-a-new-Storm-project.html) 来设置你的机器。
### Kestrel
它假设您可以在本地选择Kestrel 服务器，如上所述 [here](https://github.com/nathanmarz/storm-kestrel).

## Kestrel 服务 和 队列
单个kestrel服务器具有一组队列。Kestrel 队列是在JVM上运行的非常简单的消息队列，并使用memcache协议(具有一定的扩展名)与客户端进行通信。对于更加详细的信息，你可查看 [KestrelThriftClient](https://github.com/nathanmarz/storm-kestrel/blob/master/src/jvm/org/apache/storm/spout/KestrelThriftClient.java) 类里面提供的 [storm-kestrel](https://github.com/nathanmarz/storm-kestrel) 项目.

每个队列按照FIFO(先进先出)的原则进行严格排序。跟随性能项目缓存在系统内存中；但是，只有前128MB保存在内存中。当服务器停止时，队列状态存储在日志文件中。

此外，还可以从 [here](https://github.com/nathanmarz/kestrel/blob/master/docs/guide.md)
找到细节。

Kestrel is:
* fast
* small
* durable(耐久)
* reliable(稳定)

例如，Twitter 使用 Kestrel 作为其消息传递基础设施的骨干，如上所述[here] (http://bhavin.directi.com/notes-on-kestrel-the-open-source-twitter-queue/).

## 添加项目至 Kestrel
首先，我们需要一个可以将项目添加到Kestrel队列的程序。以下方法受益于 KestrelClient的实现 [storm-kestrel](https://github.com/nathanmarz/storm-kestrel). 它将句子添加到从包含五个可能句子的数组中随机选择的Kestrel队列中。

```
    private static void queueSentenceItems(KestrelClient kestrelClient, String queueName)
			throws ParseError, IOException {

		String[] sentences = new String[] {
	            "the cow jumped over the moon",
	            "an apple a day keeps the doctor away",
	            "four score and seven years ago",
	            "snow white and the seven dwarfs",
	            "i am at two with nature"};

		Random _rand = new Random();

		for(int i=1; i<=10; i++){

			String sentence = sentences[_rand.nextInt(sentences.length)];

			String val = "ID " + i + " " + sentence;

			boolean queueSucess = kestrelClient.queue(queueName, val);

			System.out.println("queueSucess=" +queueSucess+ " [" + val +"]");
		}
	}
```

## 将项目从 Kestrel 移除

此方法将队列中的项目排队，而不是删除。
```
    private static void dequeueItems(KestrelClient kestrelClient, String queueName) throws IOException, ParseError
			 {
		for(int i=1; i<=12; i++){

			Item item = kestrelClient.dequeue(queueName);

			if(item==null){
				System.out.println("The queue (" + queueName + ") contains no items.");
			}
			else
			{
				byte[] data = item._data;

				String receivedVal = new String(data);

				System.out.println("receivedItem=" + receivedVal);
			}
		}
```

此方法将队列中的项目排队，然后将其删除。
This method dequeues items from a queue and then removes them.
```
    private static void dequeueAndRemoveItems(KestrelClient kestrelClient, String queueName)
    throws IOException, ParseError
		 {
			for(int i=1; i<=12; i++){

				Item item = kestrelClient.dequeue(queueName);


				if(item==null){
					System.out.println("The queue (" + queueName + ") contains no items.");
				}
				else
				{
					int itemID = item._id;


					byte[] data = item._data;

					String receivedVal = new String(data);

					kestrelClient.ack(queueName, itemID);

					System.out.println("receivedItem=" + receivedVal);
				}
			}
	}
```

## 连续添加项目至 Kestrel

这是我们的最终运行程序，以便连续地将句子项添加到本地运行的Kestrel服务器的名为 **sentence_queue** 的队列中。

为了阻止它在控制台中键入一个关闭括号 char ']' ,然后按 'Enter'。

```
    import java.io.IOException;
    import java.io.InputStream;
    import java.util.Random;

    import org.apache.storm.spout.KestrelClient;
    import org.apache.storm.spout.KestrelClient.Item;
    import org.apache.storm.spout.KestrelClient.ParseError;

    public class AddSentenceItemsToKestrel {

    	/**
    	 * @param args
    	 */
    	public static void main(String[] args) {

    		InputStream is = System.in;

			char closing_bracket = ']';

			int val = closing_bracket;

			boolean aux = true;

			try {

				KestrelClient kestrelClient = null;
				String queueName = "sentence_queue";

				while(aux){

					kestrelClient = new KestrelClient("localhost",22133);

					queueSentenceItems(kestrelClient, queueName);

					kestrelClient.close();

					Thread.sleep(1000);

					if(is.available()>0){
					 if(val==is.read())
						 aux=false;
					}
				}
			} catch (IOException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			catch (ParseError e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}

			System.out.println("end");

	    }
	}
```
## 使用 KestrelSpout
该拓扑结构使用KestrelSpout从Kestrel队列中读取句子，将句子分解成其组成词(Bolt: SplitSentence),然后为每个单词发出它之前(Bolt: WordCount)所见到的次数。数据的处理方式如下所述 [Guaranteeing message processing](Guaranteeing-message-processing.html).

```
    TopologyBuilder builder = new TopologyBuilder();
    builder.setSpout("sentences", new KestrelSpout("localhost",22133,"sentence_queue",new StringScheme()));
    builder.setBolt("split", new SplitSentence(), 10)
    	        .shuffleGrouping("sentences");
    builder.setBolt("count", new WordCount(), 20)
	        .fieldsGrouping("split", new Fields("word"));
```

## 执行

首先，在生产或开发模式下启动您本地的Kestrel服务器。

等大约5秒，以避免ConnectionException。

现在执行程序将项目添加到队列并启动Storm 拓扑。启动程序的排序并不重要。

如果您使用TOPOLOGY_DEBUG运行拓扑，您应该会看到在拓扑中发出的元组。
