## Trie树

#### TrieNode在zookeeper quota结构中的应用

zookeeper中设置节点的quota信息会使用Trie树来保存，利用其优点，实现最长路径匹配，获取节点设置的quota limit

DataTree.java代码中PathTrie保存着设置的quota节点信息
``` java
     // now check if its one of the zookeeper node child
        if (parentName.startsWith(quotaZookeeper)) {
            // now check if its the limit node
            if (Quotas.limitNode.equals(childName)) {
                // this is the limit node
                // get the parent and add it to the trie
                pTrie.addPath(parentName.substring(quotaZookeeper.length()));   // 构建trie树
            }
            if (Quotas.statNode.equals(childName)) {
                updateQuotaForPath(parentName
                        .substring(quotaZookeeper.length()));
            }
        }
```
PathTrie中Trie树的结构
```java

    static class TrieNode {
        boolean property = false;
        final HashMap<String, TrieNode> children;
        TrieNode parent = null;
   }
```
添加path的操作：
```java
 /**
     * add a path to the path trie 
     * @param path
     */
    public void addPath(String path) {
        if (path == null) {
            return;
        }
        String[] pathComponents = path.split("/");
        TrieNode parent = rootNode;
        String part = null;
        if (pathComponents.length <= 1) {
            throw new IllegalArgumentException("Invalid path " + path);
        }
        for (int i=1; i<pathComponents.length; i++) {
            part = pathComponents[i];
            if (parent.getChild(part) == null) {
                parent.addChild(part, new TrieNode(parent));  //内部TrieNode的add操作
            }
            parent = parent.getChild(part);
        }
        parent.setProperty(true);
    }

```
获取最长路径前缀的操作：
```java
/**
     * return the largest prefix for the input path.
     * @param path the input path
     * @return the largest prefix for the input path.
     */
    public String findMaxPrefix(String path) {
        if (path == null) {
            return null;
        }
        if ("/".equals(path)) {
            return path;
        }
        String[] pathComponents = path.split("/");
        TrieNode parent = rootNode;
        List<String> components = new ArrayList<String>();
        if (pathComponents.length <= 1) {
            throw new IllegalArgumentException("Invalid path " + path);
        }
        int i = 1;
        String part = null;
        StringBuilder sb = new StringBuilder();
        int lastindex = -1;
        while((i < pathComponents.length)) {    // 开始匹配
            if (parent.getChild(pathComponents[i]) != null) {
                part = pathComponents[i];
                parent = parent.getChild(part);
                components.add(part);
                if (parent.getProperty()) {
                    lastindex = i-1;
                }
            }
            else {
                break;
            }
            i++;
        }
        for (int j=0; j< (lastindex+1); j++) {
            sb.append("/" + components.get(j));
        }
        return sb.toString();
    }
```
