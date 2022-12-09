# cluster.reconnect
* alertmanager->cluster.go -> Peer.reconnect
  * memeberlist -> memberlist.go -> MemberList.Join -> MemberList.pushPullNode.
* How alertmanager send nf log data to remote node?
  * where is nf log data?
* How alertmanager use nf log?
  * In DedupStage.Exec, it will query nf log.
* What is nf log?
  * Where intialization of nflog
    * main.go: notificationLog, err := nflog.New(notificationLogOpts...)
  * It will persist to file: storage.path/nflog
  * what's the format? Protocol Buffer: nflog/nflogpb/nflog.proto
* factors that impacts if needs send alert to receiver:
  * See below notification pipeline.
* how alert manager tell other peers notification has been sent?
  * The facility for gossiping notification log:
    * main.go -> run -> c := peer.AddState("nfl", notificationLog, prometheus.DefaultRegisterer)
      * It will return ClusterChannel to which alertmanager can send notification to peers.
      * It was followed by call: notificationLog.SetBroadcast(c.Broadcast)
        * So nflog will use Channel's c.Broadcast to send notification record to peers.
* Where to call the receiver?
  * notify.go -> RetryStage.exec -> r.integration.Notify(ctx, sent...)
* task: to log the nofitication gossip?
  * nflog -> Merge: level.Debug(l.logger).Log("msg", "gossiping new entry", "entry", e)
* Who will trigger the pipeline?
  * Dispatcher
  * main.go: disp = dispatch.NewDispatcher(alerts, routes, pipeline, marker, timeoutFunc, nil, logger, dispMetrics)
  * dispatcher -> Run -> run -> processAlert -> stage.Exec
* How do these stages construct the pipeline?
  * MultiStage + FanoutStage.
    * MultiStage will run Stage one by one.
    * FanoutStage will run Stages concurrently.
* How Dedup stage take effect?
  * TODO
* Which logic will use pushpull-interval?
  * main.go: cluster.Create(pushPullInterval)
  * memberlist will use memberlist.config.delegate.LocalState to get the data to do full state sync using TCP
  * alertmanager will register cluster.Peer.delegate to memberlist.config.delegate
  * alertmanager.clsuter.delegate.LocalState will serialze delegate.stats to bytes and return these bytes. Actually, 
    * delegate.stats is from Peer.stats:states map[string]State in which there will be 'nfl' and 'sil' key. In main.go,
    * peer.AddState was called for adding 'nfl' and 'sil' state.
* Which logic will use gossip-interval?
  * main.go: cluster.Create(gossipInterval)
    * gossip-interval and pushpull-interval will forward to memberlist
    * cluster.Create will return alertmanager:cluster.go:Peer.
      * Peer has memeber mlist. mlist contains pushpull-interval and gossip-interval.
      * gossip-interval and pushpull intrerval are field of mlist: memberlist.Config
      * How memberlist use it? There are comments in code. But not detailed.
      * How memberlist lib transfer data for the applicaion (e.g. AlertManager ) which is using it?
  * main.go: go peer.Settle(ctx, *gossipInterval*10)
    * peer is return value of above call cluster.Createo
    * GossipSettleStage.Exec will wait for completion of Settle.
    * In Settle, by gorupInterval*10, it will check if cluster settled by evaluating the number of Peer.Peers()

* How alertmanager tranfer its user data (nflog, silences) to memberlist?
  *  Or how memberlist transfer Alertmanager's data?
  * memberlist:delegate.go: memberlist.Delegate. 
  * AlertManager: cluster.delegate.LocateState contains AM's data FullState
    * FullState contains Parts. Parts contains data of AM
    * Peer.stats map contains all AM data? delegate construct inherits from Peer?
    * Peer.AddState to add AM data.
    * in main.go, there is below two calls to add nf log and silences to stats
      * c := peer.AddState("nfl", notificationLog, prometheus.DefaultRegisterer)
      * c := peer.AddState("sil", silences, prometheus.DefaultRegisterer)
  * Push notification log record to gossip queue
    * nflog.Log.Log()
      * Log.broadcast  (broadcast matched UDP?)
        * above callback broadcast was set in main.go: notificationLog.SetBroadcast(c.Broadcast)
* Current summary (guess):
  * When Log.log, alertmanager will push single message to queue of gossip, then memberlist will gossip message asyncly.
  * The gossip interval depends on the values of command line option --gossip-interval
  * However, gossip is using UDP, best-effort transmit. It may fail to send data to peers. So it lead to deuplicated alert.
  * Meanwhile, pushpull will use TCP to sync the full state between peers. The interval depends on cmd line option --pushpull-interval
  * What's the data size of full state?   size of silicences data and alert?
    * 
* WAN or LAN? Does AlertManager support WAN? And how?
  * AlertManager default config is for LAN.  We can change default value for WAN. Should we?
  * memberlist has DefaultWANConfig.

* How large will nfl and sil state data be?
  * TODO
  * https://github.com/prometheus/alertmanager/issues/1605
    * Just FYI: Data Retention does not affect Silences with longer duration than the data retention period.
  * Retention: retentionduration 140 hours by default.
    * nflog: log.run.GC will remove entry if the entry needs retention.
    * silences, similar function. However, the retention config is ExpireTime+Retention.
  * In our PRD env,
    * Data size for nflog is 3.0K. silences is less than 1K
  ```
   [svc_prdciqprometheus@ciqplat1prom01 alertmanager-data]$ ll -h
    total 8.0K
    -rw-r----- 1 svc_prdciqprometheus svc_prdciqprometheus 3.0K Dec  9 02:51 nflog
    -rw-r----- 1 svc_prdciqprometheus svc_prdciqprometheus  307 Dec  9 02:51 silences
  ```
  
* How memberlist pushpull?
  * MemberList.shcedule()
    * go m.pushPullTrigger(stopCh)
      * MemberList.pushPullTrigger()
        * MemberList.pushPull
          * select 1 random node
          * MemerList.pushPullNode
            * MemerberList.sendAndReceiveState
              * MemberList.sendLocalState
                * Fetch user data via m.config.Delegate.LocalState(join). 
                * send via TCP.
* How memberlist gossip?
* TODO: enable DEBUG log of memberlist to observer the behavior?

* References
  * https://promcon.io/2017-munich/slides/alertmanager-and-high-availability.pdf
  * ![notification-pipeline](./jichao_images/notification-pipeline.png)
  * ![what-gossiped](./jichao_images/what-gossiped.png)
* memberlist
  * https://medium.com/@satrobit/introduction-to-gossip-epidemic-protocol-and-memberlist-5424352cdce0