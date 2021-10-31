# flume面试题

### flume怎么保证事务的

flume以event为最小消息单元

先从source端，put消息进入channel（此时开始put事务），put的消息是putList封装

只有put动作完成后，putList才会清空

之后sink端从channel中take消息

之后当sink完成后，take事务才会完成