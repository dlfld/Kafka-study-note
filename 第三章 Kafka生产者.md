## 创建Kafka生产者

+ 要往kafka中写入数据，首先要创建一个生产者对象，并设置一些属性。

+ kafka生产者有3个必选属性。  其他的不写的话就是使用默认的方式

  + bootstrap.servers  该属性指定broker的地址清单，地址的格式为ip：port，生产者会从一个broker里面获取到其他broker的信息，不过还是建议最好写两个broker的信息，因为如果一个dawn了的情况下，还有另一个可以使用和获取信息。

  + key.serializer  broker希望接收到的消息的键和值都是字节数组。 -> 一句话 就是指定键的序列化方式而已

  + value.serializer 同上理。指定值的序列化方式而已

    ps： 如果用kafka传对象的话还要自己去实现序列化方式那些，感觉并不是很好用，推荐直接把对象转为json字符串 用string的序列化方式

+ 生产者发送消息有3种方式：

  + 发送并忘记（fire-and-forget)：把消息发送给服务器，但是并不关心它是否能够正常到达。使用这种方式有时候也会丢失一些信息。
  + 同步发送： 我们使用send() 方法发送信息，他会返回一个future对象，调用get()方法进行等待，就可以知道消息是否发送成功。
  + 异步发送：我们调用send()方法并指定一个回调函数，服务器在返回相应时调用该函数。
  + 总结区别：调用了get的是同步 不调用的是发送并忘记，有回调的是异步

## 生产者的配置

生产者还有很多可以配置的参数，他们大多数都有较为合理的默认值。所以没有必要去修改他们。不过有几个参数在内存使用、性能和可靠性方面对生产者的影响比较大。

+ acks： 这个参数指定了必须有多少个分区副本来接收消息，生产者才会认为消息是发送成功的。
  + acks=0: 生产者在成功写入消息之前不会等待任何来自服务器的响应。也就是说即使是服务器没有收到消息生产者也是无从得知的。
    + 缺点：消息很容易丢失。
    + 优点：因为生产者不需要等待服务器的响应，因此他可以以网络能够支持的最大速度发送消息，从而达到很高的吞吐量。
  + acks=1: 只要集群的首领节点收到消息，生产者就会收到一个来自服务器的成功响应。如果消息无法达到首领节点，生产者会收到一个错误响应，并且会重新发送消息。不过如果一个没有收到消息的节点成为新首领，消息还是会丢失。  **此时吞吐里取决于消息发送的方式是同步还是异步**
  + acks=all:只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。**这种模式是最安全的**，他可以保证不止一个服务器收到消息，就算有服务器发生了崩溃，整个集群仍然可以运行。
+  buffer.memory: 设置生产者内存缓冲区大小，生产者用它缓冲要发送到服务器的消息。
  + 如果应用程序发送消息的速度超过发送到服务器的速度，会导致存储空间不足。这个时候send()方法调用要么被阻塞，要么抛出异常。
+ compression.type:默认情况下，消息发送的时候不会被压缩。该参数可以设置snappy，gzip，lz4 他指定了消息被发送给broker之前使用那种压缩算法进行压缩。
+ retires：生产者从服务器收到错误有可能是临时性的错误（比如分区找不到首领）。在这种情况下，retires参数的值决定了生产者可以重发消息的次数，如果达到次数，生产者会放弃重试并返回次数。默认情况下，生产者会在每次重试之前等待100ms，不过可以通过参数**retry.backoff.ms**参数来改变这个时间间隔。
+ batch.size:有多条消息需要被发送到同一个分区的时候，生产者会把他们放到同一个批次里。该参数指定了一个批次可以使用的内存大小(按照字节计算),当批次被填满批次里面的所有消息会被发送出去。不过生产者并不一定会等到批次填满才会发送，半满的批次，甚至只包含一个消息的批次也有可能会发送。所以就算设置得很大也不会造成延迟。只是会占用更多的内存而已。设置小了的话发消息会更频繁，会增加额外的开销。
+ linger.ms:生产者在发送批次之前等待更多消息加入批次的时间。Kafkaproducer会在批次填满或linger.ms达到上限时把批次发送出去。默认情况下是有一个发送一个。如果设置了这个值的话，让生产者发送批次之前等一会儿。使得更多的消息加入批次。虽然会有延迟但是会提升吞吐量。 （一次性发送更多的消息，每个消息的开销就变小了）。
+ client.id:该参数是任意字符串。服务器用它来识别消息的来源。
+ max.in.flight.requests.per.connection:该参数指定了生产者在收到服务器响应之前可以发送多少个消息。值越高占用更多的内存，吞吐量越大。把它设置为1 可以保证消息按照发送的顺序写入服务器，即使发生了重试。
+ timeout.ms、request.timeout.ms、metadata.fetch.timeout.ms:request.timeout.ms指定了生产者在发送数据时等待服务器返回响应时间，metadata.fetch.timeout.ms指定生产者在获取元数据时等待服务器响应的时间，如果超时，要么重试发送数据，要么返回异常。timeout.ms指定了broker等待同步副本返回消息确认的时间，与acks的配置相匹配。如果指定时间内没有收到同步副本的确认，那么broker就会返回一个错误。
+ max.block.ms:该参数指定了调用send或partitionsFor方法获取元数据时生产者的阻塞时间。当生产者发送缓冲区满或者没有可用的元数据时这些方法会阻塞，在阻塞时间达到max.block.ms时生产者就会抛出异常。
+ max.requests.size:该参数用于控制生产者发送请求的大小。他可以指能发送的单个消息的最大值。也可以指单个请求里所有消息的总大小。另外broker对可接收的消息最大值也有自己的限制（message.max.bytes) 所以两边最好匹配，避免生产者发送消息被broker拒绝。
+ receive.buffer.bytes和send.buffer.bytes：这两个参数分别指定了TCP socket 接收和发送数据包的缓冲区大小。如果他们被设置为-1就使用操作系统的默认值。

## 分区

+ ProducerRecord 对象包含了目标主题、键和值。键有两个用途：1.可以作为消息的附加消息。2.可以用来决定消息应该被写到那个分区。拥有相同键的消息将被写到同一个分区。
+ 键和分区器的选择
  + 如果键被设置为null，并且使用了默认的分区器，那么记录将会被**随机发送到主题的各个可用的分区上** 分区器使用轮训算法（Round Robin）算法将消息均衡的分部到各个分区上。
  + 如果键不为空，并且使用了默认的分区器，那么kafka会对键进行散列，然后根据散列值把消息映射到特定的分区上。这里的关键是**同一个键总是被映射到同一个分区上**

