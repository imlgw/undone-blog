---
title: 
   数据结构：Trie
tags: 
  [Trie,算法]
categories:
    [算法]
---

## [835. Trie字符串统计](https://www.acwing.com/problem/content/description/837/)
维护一个字符串集合，支持两种操作：

- `I x` 向集合中插入一个字符串$x$；
- `Q x` 询问一个字符串在集合中出现了多少次。
共有$N$个操作，输入的字符串总长度不超过$10^5$，字符串仅包含小写英文字母。

**输入格式**

第一行包含整数$N$，表示操作数。

接下来$N$行，每行包含一个操作指令，指令为 `I x` 或 `Q x` 中的一种。

**输出格式**

对于每个询问指令`Q x`，都要输出一个整数作为结果，表示$x$在集合中出现的次数。每个结果占一行。

**数据范围**：$1≤N≤2∗104$

**输入样例：**
```c
5
I abc
Q abc
Q ab
I ab
Q ab
```
**输出样例：**
```c
1
0
1
```

### 解法一
模板题，一开始写的利用类创建节点构建字典树的方法
```java
import java.util.*;
import java.io.*;

class Main {

    static Node root;

    static class Node {
        Node[] next;
        int cnt;
        public Node() {
            next = new Node[26];
        }
    }
    
    public static void insert(String str) {
        Node cur = root;
        int i = 0;
        while (i < str.length()) {
            int c = str.charAt(i++)-'a';
            if (cur.next[c] == null) {
                cur.next[c] = new Node();
            }
            cur = cur.next[c];
        }
        cur.cnt++;
    }

    public static int query(String str) {
        Node cur = root;
        int i = 0;
        while (i < str.length()) {
            int c = str.charAt(i)-'a';
            if (cur.next[c]==null) return 0;
            cur = cur.next[c];
            i++;
        }
        return cur.cnt;
    }

    public static void main(String... args) throws Exception {
        PrintWriter out = new PrintWriter(new BufferedOutputStream(System.out));
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        // BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("./input.txt")));
        int N = Integer.valueOf(br.readLine());
        root = new Node();
        while (N-- > 0) {
            String[] op = read(br);
            if ("I".equals(op[0])) {
                insert(op[1]);
            } else {
                out.println(query(op[1]));
            }
        }
        out.flush();
    }

    public static String[] read(BufferedReader br) throws Exception {
        return Arrays.stream(br.readLine().split(" ")).toArray(String[]::new);
    }
}
```
### 解法二
看了题解发现可以用数组模拟，实现了下，其实也很好理解，$son[i][j]$代表第$i$个节点的第$j$号字符的节点编号，如果节点编号为$0$说明该字符不存在。$idx$就是节点编号，最坏情况就是全部字符串成一条链，那么$idx$最大就是$1e5$
```java
import java.util.*;
import java.io.*;

class Main {

    static int idx;
    static int[][] son;
    static int[] cnt;
    
    public static void insert(String str) {
        int cur = 0;
        for (int i = 0; i < str.length(); i++) {
            int c = str.charAt(i)-'a';
            if (son[cur][c] == 0) {
                son[cur][c] = ++idx;
            }
            cur = son[cur][c];
        }
        cnt[cur]++;
    }

    public static int query(String str) {
        int cur = 0;
        for (int i = 0; i < str.length(); i++) {
            int c = str.charAt(i)-'a';
            if (son[cur][c]==0) return 0;
            cur = son[cur][c];
        }
        return cnt[cur];
    }

    public static void main(String... args) throws Exception {
        PrintWriter out = new PrintWriter(new BufferedOutputStream(System.out));
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        // BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("./input.txt")));
        int MAX = (int)1e5+10;
        son = new int[MAX][26]; cnt = new int[MAX];
        int N = Integer.valueOf(br.readLine());
        while (N-- > 0) {
            String[] op = read(br);
            if ("I".equals(op[0])) {
                insert(op[1]);
            } else {
                out.println(query(op[1]));
            }
        }
        out.flush();
    }

    public static String[] read(BufferedReader br) throws Exception {
        return Arrays.stream(br.readLine().split(" ")).toArray(String[]::new);
    }
}
```