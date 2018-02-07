---
layout: post
title: Zookeeper源码分析zab协议
categories: [编程, java]
tags: [zab, zookeeper]
---

#### 1. 简介

`Zookeeper`:ZooKeeper是一个分布式协调服务，可实现配置管理、命名服务、分布式锁、选举等服务

`zab`:ZooKeeper Atomic Broadcast, 原子广播协议(也有算法一说)，用于保障数据一的致性

#### 2. 启动类
在zookeeper启动脚本中找到启动类：

bin/zkServer.sh
```
ZOOMAIN="org.apache.zookeeper.server.quorum.QuorumPeerMain"
```

#### 3. 源码分析

##### 3.1 main函数

QuorumPeerMain.main()
```java
public static void main(String[] args) {
    QuorumPeerMain main = new QuorumPeerMain();
    try {
        main.initializeAndRun(args);
    }
    ...
}
```

QuorumPeerMain.initializeAndRun()
```java
QuorumPeerConfig config = new QuorumPeerConfig();
if (args.length == 1) {
    config.parse(args[0]);
}

// Start and schedule the the purge task
DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
        .getDataDir(), config.getDataLogDir(), config
        .getSnapRetainCount(), config.getPurgeInterval());
purgeMgr.start();

if (args.length == 1 && config.servers.size() > 0) {
    runFromConfig(config);
} else {
    LOG.warn("Either no config or no quorum defined in config, running "
            + " in standalone mode");
    // there is only server in the quorum -- run as standalone
    ZooKeeperServerMain.main(args);
}
```

> server配置时以集群模式运行，否则以单机模式运行，这里忽略单机模式


##### 3.2 配置解析

QuorumPeerConfig.parse()
```java
} else if (key.startsWith("server.")) {
    int dot = key.indexOf('.');
    long sid = Long.parseLong(key.substring(dot + 1));
    String parts[] = splitWithLeadingHostname(value);
    if ((parts.length != 2) && (parts.length != 3) && (parts.length !=4)) {
        LOG.error(value
           + " does not have the form host:port or host:port:port " +
           " or host:port:port:type");
    }
    LearnerType type = null;
    String hostname = parts[0];
    Integer port = Integer.parseInt(parts[1]);
    Integer electionPort = null;
    if (parts.length > 2){
        electionPort=Integer.parseInt(parts[2]);
    }
    if (parts.length > 3){
        if (parts[3].toLowerCase().equals("observer")) {
            type = LearnerType.OBSERVER;
        } else if (parts[3].toLowerCase().equals("participant")) {
            type = LearnerType.PARTICIPANT;
        } else {
            throw new ConfigException("Unrecognised peertype: " + value);
        }
    }
    if (type == LearnerType.OBSERVER){
        observers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
    } else {
        servers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
    }

    ...
```

> server配置项key的格式为`server.id`，其中`id`必须为`long`类型
> server值的格式为`host:port[[:port]:type]`，第一个port用于事务分发，第二个port用于选举，type可以指定为`observer`或`participant`

##### 3.3 QuorumPeer线程

QuorumPeer.start()
```java
public synchronized void start() {
    loadDataBase();
    cnxnFactory.start();
    startLeaderElection();
    super.start();
}
```

QuorumPeer.loadDataBase()
```java
File updating = new File(getTxnFactory().getSnapDir(),
                                 UPDATING_EPOCH_FILENAME);
try {
    zkDb.loadDataBase();

    // load the epochs
    long lastProcessedZxid = zkDb.getDataTree().lastProcessedZxid;
    long epochOfZxid = ZxidUtils.getEpochFromZxid(lastProcessedZxid);
    try {
        currentEpoch = readLongFromFile(CURRENT_EPOCH_FILENAME);
        if (epochOfZxid > currentEpoch && updating.exists()) {
            LOG.info("{} found. The server was terminated after " +
                     "taking a snapshot but before updating current " +
                     "epoch. Setting current epoch to {}.",
                     UPDATING_EPOCH_FILENAME, epochOfZxid);
            setCurrentEpoch(epochOfZxid);
            if (!updating.delete()) {
                throw new IOException("Failed to delete " +
                                      updating.toString());
            }
        }
    } catch(FileNotFoundException e) {
        // pick a reasonable epoch number
        // this should only happen once when moving to a
        // new code version
        currentEpoch = epochOfZxid;
        LOG.info(CURRENT_EPOCH_FILENAME
                + " not found! Creating with a reasonable default of {}. This should only happen when you are upgrading your installation",
                currentEpoch);
        writeLongToFile(CURRENT_EPOCH_FILENAME, currentEpoch);
    }
    if (epochOfZxid > currentEpoch) {
        throw new IOException("The current epoch, " + ZxidUtils.zxidToString(currentEpoch) + ", is older than the last zxid, " + lastProcessedZxid);
    }
    try {
        acceptedEpoch = readLongFromFile(ACCEPTED_EPOCH_FILENAME);
    } catch(FileNotFoundException e) {
        // pick a reasonable epoch number
        // this should only happen once when moving to a
        // new code version
        acceptedEpoch = epochOfZxid;
        LOG.info(ACCEPTED_EPOCH_FILENAME
                + " not found! Creating with a reasonable default of {}. This should only happen when you are upgrading your installation",
                acceptedEpoch);
        writeLongToFile(ACCEPTED_EPOCH_FILENAME, acceptedEpoch);
    }
    if (acceptedEpoch < currentEpoch) {
        throw new IOException("The accepted epoch, " + ZxidUtils.zxidToString(acceptedEpoch) + " is less than the current epoch, " + ZxidUtils.zxidToString(currentEpoch));
    }
} catch(IOException ie) {
    LOG.error("Unable to load database on disk", ie);
    throw new RuntimeException("Unable to run quorum server ", ie);
}
```

> `zxid`:zookeeper事务id，用64位数字表示，其中高32位为`Leader`的`epoch`，低32位为递增序号
> `epoch`:时期，纪元，用`zxid`的前32位表示，随着`Leader`的更换，`epoch`也会递增

QuorumPeer.startLeaderElection()
```java
try {
    currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
} catch(IOException e) {
    RuntimeException re = new RuntimeException(e.getMessage());
    re.setStackTrace(e.getStackTrace());
    throw re;
}
for (QuorumServer p : getView().values()) {
    if (p.id == myid) {
        myQuorumAddr = p.addr;
        break;
    }
}
if (myQuorumAddr == null) {
    throw new RuntimeException("My id " + myid + " not in the peer list");
}
if (electionType == 0) {
    try {
        udpSocket = new DatagramSocket(myQuorumAddr.getPort());
        responder = new ResponderThread();
        responder.start();
    } catch (SocketException e) {
        throw new RuntimeException(e);
    }
}
this.electionAlg = createElectionAlgorithm(electionType);
```
> 用自己的`zxid`和`epoch`发起投票，选自己

QuorumPeer.createElectionAlgorithm
```java
Election le=null;

//TODO: use a factory rather than a switch
switch (electionAlgorithm) {
case 0:
    le = new LeaderElection(this);
    break;
case 1:
    le = new AuthFastLeaderElection(this);
    break;
case 2:
    le = new AuthFastLeaderElection(this, true);
    break;
case 3:
    qcm = createCnxnManager();
    QuorumCnxManager.Listener listener = qcm.listener;
    if(listener != null){
        listener.start();
        le = new FastLeaderElection(this, qcm);
    } else {
        LOG.error("Null listener when initializing cnx manager");
    }
    break;
```

> `electionType`默认是`3`，所以默认选举策略为`FastLeaderElection`

QuorumPeer.run()
```java
while (running) {
    switch (getPeerState()) {
    case LOOKING:
        LOG.info("LOOKING");

        if (Boolean.getBoolean("readonlymode.enabled")) {
            LOG.info("Attempting to start ReadOnlyZooKeeperServer");

            // Create read-only server but don't start it immediately
            final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(
                    logFactory, this,
                    new ZooKeeperServer.BasicDataTreeBuilder(),
                    this.zkDb);

            // Instead of starting roZk immediately, wait some grace
            // period before we decide we're partitioned.
            //
            // Thread is used here because otherwise it would require
            // changes in each of election strategy classes which is
            // unnecessary code coupling.
            Thread roZkMgr = new Thread() {
                public void run() {
                    try {
                        // lower-bound grace period to 2 secs
                        sleep(Math.max(2000, tickTime));
                        if (ServerState.LOOKING.equals(getPeerState())) {
                            roZk.startup();
                        }
                    } catch (InterruptedException e) {
                        LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                    } catch (Exception e) {
                        LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                    }
                }
            };
            try {
                roZkMgr.start();
                setBCVote(null);
                setCurrentVote(makeLEStrategy().lookForLeader());
            } catch (Exception e) {
                LOG.warn("Unexpected exception",e);
                setPeerState(ServerState.LOOKING);
            } finally {
                // If the thread is in the the grace period, interrupt
                // to come out of waiting.
                roZkMgr.interrupt();
                roZk.shutdown();
            }
        } else {
            try {
                setBCVote(null);
                setCurrentVote(makeLEStrategy().lookForLeader());
            } catch (Exception e) {
                LOG.warn("Unexpected exception", e);
                setPeerState(ServerState.LOOKING);
            }
        }
        break;
    case OBSERVING:
        try {
            LOG.info("OBSERVING");
            setObserver(makeObserver(logFactory));
            observer.observeLeader();
        } catch (Exception e) {
            LOG.warn("Unexpected exception",e );
        } finally {
            observer.shutdown();
            setObserver(null);
            setPeerState(ServerState.LOOKING);
        }
        break;
    case FOLLOWING:
        try {
            LOG.info("FOLLOWING");
            setFollower(makeFollower(logFactory));
            follower.followLeader();
        } catch (Exception e) {
            LOG.warn("Unexpected exception",e);
        } finally {
            follower.shutdown();
            setFollower(null);
            setPeerState(ServerState.LOOKING);
        }
        break;
    case LEADING:
        LOG.info("LEADING");
        try {
            setLeader(makeLeader(logFactory));
            leader.lead();
            setLeader(null);
        } catch (Exception e) {
            LOG.warn("Unexpected exception",e);
        } finally {
            if (leader != null) {
                leader.shutdown("Forcing shutdown");
                setLeader(null);
            }
            setPeerState(ServerState.LOOKING);
        }
        break;
    }
}
```

> 在`run`方法中进行事件循环

##### 3.4 LOOKING

```java
setCurrentVote(makeLEStrategy().lookForLeader());
```

> 节点启动时的初始状态

FastLeaderElection.lookForLeader()
```java
HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

int notTimeout = finalizeWait;

synchronized(this){
    logicalclock++;
    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}

LOG.info("New election. My id =  " + self.getId() +
        ", proposed zxid=0x" + Long.toHexString(proposedZxid));
sendNotifications();

/*
 * Loop in which we exchange notifications until we find a leader
 */

while ((self.getPeerState() == ServerState.LOOKING) &&
        (!stop)){
    /*
     * Remove next notification from queue, times out after 2 times
     * the termination time
     */
    Notification n = recvqueue.poll(notTimeout,
            TimeUnit.MILLISECONDS);

    /*
     * Sends more notifications if haven't received enough.
     * Otherwise processes new notification.
     */
    if(n == null){
        if(manager.haveDelivered()){
            sendNotifications();
        } else {
            manager.connectAll();
        }

        /*
         * Exponential backoff
         */
        int tmpTimeOut = notTimeout*2;
        notTimeout = (tmpTimeOut < maxNotificationInterval?
                tmpTimeOut : maxNotificationInterval);
        LOG.info("Notification time out: " + notTimeout);
    }
    else if(self.getVotingView().containsKey(n.sid)) {
        /*
         * Only proceed if the vote comes from a replica in the
         * voting view.
         */
        switch (n.state) {
        case LOOKING:
            // If notification > current, replace and send messages out
            if (n.electionEpoch > logicalclock) {
                logicalclock = n.electionEpoch;
                recvset.clear();
                if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                        getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                    updateProposal(n.leader, n.zxid, n.peerEpoch);
                } else {
                    updateProposal(getInitId(),
                            getInitLastLoggedZxid(),
                            getPeerEpoch());
                }
                sendNotifications();
            } else if (n.electionEpoch < logicalclock) {
                if(LOG.isDebugEnabled()){
                    LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                            + Long.toHexString(n.electionEpoch)
                            + ", logicalclock=0x" + Long.toHexString(logicalclock));
                }
                break;
            } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                    proposedLeader, proposedZxid, proposedEpoch)) {
                updateProposal(n.leader, n.zxid, n.peerEpoch);
                sendNotifications();
            }

            if(LOG.isDebugEnabled()){
                LOG.debug("Adding vote: from=" + n.sid +
                        ", proposed leader=" + n.leader +
                        ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                        ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
            }

            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

            if (termPredicate(recvset,
                    new Vote(proposedLeader, proposedZxid,
                            logicalclock, proposedEpoch))) {

                // Verify if there is any change in the proposed leader
                while((n = recvqueue.poll(finalizeWait,
                        TimeUnit.MILLISECONDS)) != null){
                    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                            proposedLeader, proposedZxid, proposedEpoch)){
                        recvqueue.put(n);
                        break;
                    }
                }

                /*
                 * This predicate is true once we don't read any new
                 * relevant message from the reception queue
                 */
                if (n == null) {
                    self.setPeerState((proposedLeader == self.getId()) ?
                            ServerState.LEADING: learningState());

                    Vote endVote = new Vote(proposedLeader,
                                            proposedZxid,
                                            logicalclock,
                                            proposedEpoch);
                    leaveInstance(endVote);
                    return endVote;
                }
            }
            break;
        case OBSERVING:
            LOG.debug("Notification from observer: " + n.sid);
            break;
        case FOLLOWING:
        case LEADING:
            /*
             * Consider all notifications from the same epoch
             * together.
             */
            if(n.electionEpoch == logicalclock){
                recvset.put(n.sid, new Vote(n.leader,
                                              n.zxid,
                                              n.electionEpoch,
                                              n.peerEpoch));

                if(ooePredicate(recvset, outofelection, n)) {
                    self.setPeerState((n.leader == self.getId()) ?
                            ServerState.LEADING: learningState());

                    Vote endVote = new Vote(n.leader,
                            n.zxid,
                            n.electionEpoch,
                            n.peerEpoch);
                    leaveInstance(endVote);
                    return endVote;
                }
            }

            /*
             * Before joining an established ensemble, verify
             * a majority is following the same leader.
             */
            outofelection.put(n.sid, new Vote(n.version,
                                                n.leader,
                                                n.zxid,
                                                n.electionEpoch,
                                                n.peerEpoch,
                                                n.state));

            if(ooePredicate(outofelection, outofelection, n)) {
                synchronized(this){
                    logicalclock = n.electionEpoch;
                    self.setPeerState((n.leader == self.getId()) ?
                            ServerState.LEADING: learningState());
                }
                Vote endVote = new Vote(n.leader,
                                        n.zxid,
                                        n.electionEpoch,
                                        n.peerEpoch);
                leaveInstance(endVote);
                return endVote;
            }
            break;
        default:
            LOG.warn("Notification state unrecognized: {} (n.state), {} (n.sid)",
                    n.state, n.sid);
            break;
        }
    } else {
        LOG.warn("Ignoring notification from non-cluster member " + n.sid);
    }
}
return null;
```

> `updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch())`，选举的第一步，提议自己当`Leader`，并发送给其它节点`sendNotifications()`
> 进入事件循环`while ((self.getPeerState() == ServerState.LOOKING) && (!stop)){`
> 从接收通知队列取出收到的通知`Notification n = recvqueue.poll(notTimeout, TimeUnit.MILLISECONDS);`并根据通知节点的状态做不同处理

###### 3.4.1 LOOKEING状态的投票者

比较自己和他人的投票 FastLeaderElection.totalOrderPredicate()
```java
/*
 * We return true if one of the following three cases hold:
 * 1- New epoch is higher
 * 2- New epoch is the same as current epoch, but new zxid is higher
 * 3- New epoch is the same as current epoch, new zxid is the same
 *  as current zxid, but server id is higher.
 */

return ((newEpoch > curEpoch) ||
        ((newEpoch == curEpoch) &&
        ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
```

> 选票比较规则`epoch->zxid->sid`，如果比较胜出则忽略，否则改投对方投票

判断是否可以结束选举 FastLeaderElection.termPredicate()
```java
HashSet<Long> set = new HashSet<Long>();

/*
 * First make the views consistent. Sometimes peers will have
 * different zxids for a server depending on timing.
 */
for (Map.Entry<Long,Vote> entry : votes.entrySet()) {
    if (vote.equals(entry.getValue())){
        set.add(entry.getKey());
    }
}

return self.getQuorumVerifier().containsQuorum(set);
```

默认的`QuorumVerifier`实现为`QuorumMaj`
```java
public boolean containsQuorum(HashSet<Long> set){
    return (set.size() > half);
}
```
> `Leader`的判定规则为其选票大于参选人数的一半
> `QuorumVerifier`的另一个实现为`QuorumHierarchical`可以根据分组和权限实现更为复杂的判定规则

###### 3.4.2 OBSERVING状态的投票者
```java
case OBSERVING:
LOG.debug("Notification from observer: " + n.sid);
break;
```
> 直接忽略，OBSERVER节点无选举权

###### 3.4.3 FOLLOWING和LEADING状态的投票者

```java
case FOLLOWING:
case LEADING:
    /*
     * Consider all notifications from the same epoch
     * together.
     */
    if(n.electionEpoch == logicalclock){
        recvset.put(n.sid, new Vote(n.leader,
                                      n.zxid,
                                      n.electionEpoch,
                                      n.peerEpoch));

        if(ooePredicate(recvset, outofelection, n)) {
            self.setPeerState((n.leader == self.getId()) ?
                    ServerState.LEADING: learningState());

            Vote endVote = new Vote(n.leader,
                    n.zxid,
                    n.electionEpoch,
                    n.peerEpoch);
            leaveInstance(endVote);
            return endVote;
        }
    }
```

> 对于`FOLLOWING`和`LEADING`状态的投票者，其投票应该是一样的
> 一旦发现有确定的`Leader`，则更新自己的状态(`LEADING|OBSERVING|FOLLOWING`),投票结束

##### 3.5 OBSERVING
```java
setObserver(makeObserver(logFactory));
observer.observeLeader();
```

Observer.observeLeader()
```java
QuorumServer leaderServer = findLeader();
LOG.info("Observing " + leaderServer.addr);
try {
    connectToLeader(leaderServer.addr, leaderServer.hostname);
    long newLeaderZxid = registerWithLeader(Leader.OBSERVERINFO);

    syncWithLeader(newLeaderZxid);
    QuorumPacket qp = new QuorumPacket();
    while (this.isRunning()) {
        readPacket(qp);
        processPacket(qp);
    }
}
```

> 连接到`Leader`并接收命令，命令类型如下

Observer.processPacket()
```java
case Leader.PING:
    ping(qp);
    break;
case Leader.PROPOSAL:
    LOG.warn("Ignoring proposal");
    break;
case Leader.COMMIT:
    LOG.warn("Ignoring commit");
    break;
case Leader.UPTODATE:
    LOG.error("Received an UPTODATE message after Observer started");
    break;
case Leader.REVALIDATE:
    revalidate(qp);
    break;
case Leader.SYNC:
    ((ObserverZooKeeperServer)zk).sync();
    break;
case Leader.INFORM:
```

##### 3.6 FOLLOWING
```java
setFollower(makeFollower(logFactory));
follower.followLeader();
```

Follower.followLeader()
```java
QuorumServer leaderServer = findLeader();
try {
    connectToLeader(leaderServer.addr, leaderServer.hostname);
    long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);

    //check to see if the leader zxid is lower than ours
    //this should never happen but is just a safety check
    long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
    if (newEpoch < self.getAcceptedEpoch()) {
        LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
        throw new IOException("Error: Epoch of leader is lower");
    }
    syncWithLeader(newEpochZxid);
    QuorumPacket qp = new QuorumPacket();
    while (this.isRunning()) {
        readPacket(qp);
        processPacket(qp);
    }
}
```

> `FOLLOWING`状态节点同`OBSERVING`节点，也是连接到`Leader`并接收命令

##### 3.7 LEADING
```java
setLeader(makeLeader(logFactory));
leader.lead();
setLeader(null);
```

Leader.lead()
```java
self.tick = 0;
zk.loadData();

leaderStateSummary = new StateSummary(self.getCurrentEpoch(), zk.getLastProcessedZxid());

// Start thread that waits for connection requests from
// new followers.
cnxAcceptor = new LearnerCnxAcceptor();
cnxAcceptor.start();

readyToStart = true;
long epoch = getEpochToPropose(self.getId(), self.getAcceptedEpoch());

zk.setZxid(ZxidUtils.makeZxid(epoch, 0));

synchronized(this){
    lastProposed = zk.getZxid();
}

newLeaderProposal.packet = new QuorumPacket(NEWLEADER, zk.getZxid(),
        null, null);


if ((newLeaderProposal.packet.getZxid() & 0xffffffffL) != 0) {
    LOG.info("NEWLEADER proposal has Zxid of "
            + Long.toHexString(newLeaderProposal.packet.getZxid()));
}

waitForEpochAck(self.getId(), leaderStateSummary);
self.setCurrentEpoch(epoch);

// We have to get at least a majority of servers in sync with
// us. We do this by waiting for the NEWLEADER packet to get
// acknowledged
try {
    waitForNewLeaderAck(self.getId(), zk.getZxid(), LearnerType.PARTICIPANT);
} catch (InterruptedException e) {
    shutdown("Waiting for a quorum of followers, only synced with sids: [ "
            + getSidSetString(newLeaderProposal.ackSet) + " ]");
    HashSet<Long> followerSet = new HashSet<Long>();
    for (LearnerHandler f : learners)
        followerSet.add(f.getSid());

    if (self.getQuorumVerifier().containsQuorum(followerSet)) {
        LOG.warn("Enough followers present. "
                + "Perhaps the initTicks need to be increased.");
    }
    Thread.sleep(self.tickTime);
    self.tick++;
    return;
}

startZkServer();

/**
 * WARNING: do not use this for anything other than QA testing
 * on a real cluster. Specifically to enable verification that quorum
 * can handle the lower 32bit roll-over issue identified in
 * ZOOKEEPER-1277. Without this option it would take a very long
 * time (on order of a month say) to see the 4 billion writes
 * necessary to cause the roll-over to occur.
 *
 * This field allows you to override the zxid of the server. Typically
 * you'll want to set it to something like 0xfffffff0 and then
 * start the quorum, run some operations and see the re-election.
 */
String initialZxid = System.getProperty("zookeeper.testingonly.initialZxid");
if (initialZxid != null) {
    long zxid = Long.parseLong(initialZxid);
    zk.setZxid((zk.getZxid() & 0xffffffff00000000L) | zxid);
}

if (!System.getProperty("zookeeper.leaderServes", "yes").equals("no")) {
    self.cnxnFactory.setZooKeeperServer(zk);
}
// Everything is a go, simply start counting the ticks
// WARNING: I couldn't find any wait statement on a synchronized
// block that would be notified by this notifyAll() call, so
// I commented it out
//synchronized (this) {
//    notifyAll();
//}
// We ping twice a tick, so we only update the tick every other
// iteration
boolean tickSkip = true;

while (true) {
    Thread.sleep(self.tickTime / 2);
    if (!tickSkip) {
        self.tick++;
    }
    HashSet<Long> syncedSet = new HashSet<Long>();

    // lock on the followers when we use it.
    syncedSet.add(self.getId());

    for (LearnerHandler f : getLearners()) {
        // Synced set is used to check we have a supporting quorum, so only
        // PARTICIPANT, not OBSERVER, learners should be used
        if (f.synced() && f.getLearnerType() == LearnerType.PARTICIPANT) {
            syncedSet.add(f.getSid());
        }
        f.ping();
    }

    // check leader running status
    if (!this.isRunning()) {
        shutdown("Unexpected internal error");
        return;
    }

  if (!tickSkip && !self.getQuorumVerifier().containsQuorum(syncedSet)) {
    //if (!tickSkip && syncedCount < self.quorumPeers.size() / 2) {
        // Lost quorum, shutdown
        shutdown("Not sufficient followers synced, only synced with sids: [ "
                + getSidSetString(syncedSet) + " ]");
        // make sure the order is the same!
        // the leader goes to looking
        return;
  }
  tickSkip = !tickSkip;
}
```

> `LEADING`状态节点启动监听服务，接收`Learner`的连接请求，并发送心跳

#### 4. 参考文档
[Apache ZooKeeper](https://zookeeper.apache.org)

[ZooKeeper’s atomic broadcast protocol: Theory and practice](http://www.tcs.hut.fi/Studies/T-79.5001/reports/2012-deSouzaMedeiros.pdf)

[Apache ZAB—Zookeeper Atomic Broadcast Protocol](https://www.ixiacom.com/company/blog/apache-zab—zookeeper-atomic-broadcast-protocol)
