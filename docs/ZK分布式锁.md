``` java
package com.xiaosa.demo;

import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.ZkClient;

import java.util.List;
import java.util.stream.Collectors;


/**
 * @Description:
 * @author: 潇洒
 * @create: 2020-04-02 19:04
 */
public class DistriuteLock {


    /**
     * zk的客服端
     */
    ZkClient  zkClient;


    public DistriuteLock(){
        //连接zk
        zkClient = new ZkClient("127.0.0.1:2181", 3000);
        //判断是否有根结点
        boolean exists = zkClient.exists("/lock");
        if(!exists){
            //创建根节点
            zkClient.createPersistent("/lock");
        }
    }


    /**
     * 加锁
     */
    public void lock(){
        //创建临时结点
        Node node = creatNode();
        if(!getLock(node)){
            //如果没有获得了锁，则阻塞
            synchronized (node){
                try {
                    node.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }




    /**
     * 解锁
     */
    public void unLock(Node node){
        zkClient.delete(node.path);
    }




    /**
     *  获得锁
     */
    public Boolean getLock(Node node){
        Boolean isLock = false;
        //获得所有的临时结点
        List<String> children = zkClient.getChildren("lock")
                .stream()
                .sorted()
                .map(n -> "lock"+n)
                .collect(Collectors.toList());
        //获得最小的阶段
        String firstPath = children.get(0);
        if(firstPath.equals(node.getPath())){
            isLock = true;
        }else {
            //获取前一个结点
            String prePath = children.get(children.indexOf(node.getPath())-1);
            //前结点的事件监听
            zkClient.subscribeDataChanges(prePath, new IZkDataListener() {
                @Override
                public void handleDataChange(String s, Object o) throws Exception {
                }
                @Override
                public void handleDataDeleted(String s) throws Exception {
                    //前一个结点删除
                    synchronized (node){
                        node.notify();
                    }
                    //删除监听的事件
                    zkClient.unsubscribeDataChanges(prePath,this);
                }
            });
        }
        return isLock;
    }




    /**
     * 创建锁
     */
    public Node creatNode(){
        //创建临时结点，返回临时结点路径
        String path = zkClient.createEphemeralSequential("/lock/node", "N");
        Node node = new Node(path);
        return node;
    }


    /**
     * zk上的临时结点
     */
    class Node{
         private  String path;

        public Node(String path) {
            this.path = path;
        }

        public String getPath() {
            return path;
        }

        public void setPath(String path) {
            this.path = path;
        }
    }
}

```

