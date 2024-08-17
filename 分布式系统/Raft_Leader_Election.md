基于状态转换的事件驱动模型

```go
select {
	case<-follower:
    go func(){
        while(true){
            if(心跳间隔检测不通过){
                switch to candidate 
                return
            }
        }
    }
    case<-candidate:
    go func(){
        while(true){
            选举
            if(成功){
                switch to leader
            }else{
                time.Sleep()
            }
    	}
    }
    case<-leader:
}
```

这样做会导致起太多的go routine，而角色切换是一个常态，如果一个peer刚刚竞选成为leader，上一个未运行完的go routine又可能将其置为candidate。

长期运行的go routine

```go
// 长期运行的选举和心跳检测
func (rf *Raft) heartbeatCheck(){
	for{
		rf.mu.Lock()
		if rf.role != leader {
			rf.nonLeaderCond.Wait()
		} else {
			elapseTime := time.Now().UnixNano() - rf.latestIssueTime
			if int(elapseTime/int64(time.Millisecond)) >= rf.heartbeatPeriod {
				rf.heartbeatPeriodChan <- true
			}
		}
		rf.mu.Unlock()
		time.Sleep(time.Millisecond * 10)
	}
}

func (rf *Raft) electionTimeoutCheck(){
	for{
		rf.mu.Lock()
		if rf.role == leader {
			rf.leaderCond.Wait()
		} else {
			elapseTime := time.Now().UnixNano() - rf.latestHeardTime
			if int(elapseTime/int64(time.Millisecond)) >= rf.electionTimeout {
				rf.electionTimeoutChan <- true
			}
		}
		rf.mu.Unlock()
		time.Sleep(time.Millisecond * 10)
	}
}

// 基于事件驱动的election和heartbeat
func (rf *Raft) ticker() {
	for rf.killed() == false {
		// eventLoop
		select{
			case <- rf.electionTimeoutChan:
				go rf.election()
			case <- rf.heartbeatPeriodChan:
				go rf.sendLog()
		}
	}
}
```

日志复制

leader选举出来之后，通过`AppendEntries` RPC将日志同步给从节点

`AppendEntries`参数

```go
AppendEntriesArgs {
    Term: term,											// 当前任期
    LeaderId : rf.me,									// leaderId
    PrevLogIndex : rf.nextIndex[server]-1,				// nextIndex-1
    PrevLogTerm : rf.Log[rf.nextIndex[server]-1].Term,	// prevLogIndex日志对应的term
    Entries: logs,										// 日志，Log[nextIndex, lastLogIndex)
    LeaderCommit : commitIndex,							// leader已提交的commitIndex
}
```

一致性检查

从节点收到主节点的RPC之后，会进行一致性检查，一致性检查就是要检查从节点日志中`prevLogIndex`位置的`term`和参数中的`term`是否一致，如果

- 一致，将日志加入到从节点中，并回复成功，主节点收到半数以上的成功之后，提升`commitIndex`，同时向从节点发送RPC请求，令其更新`commitIndex`
- 不一致，拒绝这个请求，主节点收到拒绝之后，将nextIndex-1，preLogIndex=nextIndex-1，继续发送`RPC`给从节点，直到通过一致性检查

