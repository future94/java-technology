典型应用场景：[zookeeper](https://zookeeper.apache.org/)

与Raft算法类似，Raft叫term，而Zab叫epoch。心跳方向，Raft是leader发给folloer，Zab是follower给leader。