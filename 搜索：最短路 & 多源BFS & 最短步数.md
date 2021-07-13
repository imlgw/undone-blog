---
title: 
   搜索：最短路 & 多源BFS & 最短步数
tags: 
  [搜索, 算法, 最短路]
categories:
    [算法]
---
# 最短路模型
## [188. 武士风度的牛](https://www.acwing.com/problem/content/190/)

农民 John 有很多牛，他想交易其中一头被 Don 称为 The Knight 的牛。

这头牛有一个独一无二的超能力，在农场里像 Knight 一样地跳（就是我们熟悉的象棋中马的走法）。

虽然这头神奇的牛不能跳到树上和石头上，但是它可以在牧场上随意跳，我们把牧场用一个$x，y$的坐标图来表示。

这头神奇的牛像其它牛一样喜欢吃草，给你一张地图，上面标注了 The Knight 的开始位置，树、灌木、石头以及其它障碍的位置，除此之外还有一捆草。

现在你的任务是，确定 The Knight 要想吃到草，至少需要跳多少次。

The Knight 的位置用`K`来标记，障碍的位置用`*`来标记，草的位置用`H` 来标记。

这里有一个地图的例子：
```
             11 | . . . . . . . . . .
             10 | . . . . * . . . . . 
              9 | . . . . . . . . . . 
              8 | . . . * . * . . . . 
              7 | . . . . . . . * . . 
              6 | . . * . . * . . . H 
              5 | * . . . . . . . . . 
              4 | . . . * . . . * . . 
              3 | . K . . . . . . . . 
              2 | . . . * . . . . . * 
              1 | . . * . . . . * . . 
              0 ----------------------
                                    1 
                0 1 2 3 4 5 6 7 8 9 0
```
The Knight 可以按照下图中的 A,B,C,D… 这条路径用 5 次跳到草的地方（有可能其它路线的长度也是 5）：
```
             11 | . . . . . . . . . .
             10 | . . . . * . . . . .
              9 | . . . . . . . . . .
              8 | . . . * . * . . . .
              7 | . . . . . . . * . .
              6 | . . * . . * . . . F<
              5 | * . B . . . . . . .
              4 | . . . * C . . * E .
              3 | .>A . . . . D . . .
              2 | . . . * . . . . . *
              1 | . . * . . . . * . .
              0 ----------------------
                                    1
                0 1 2 3 4 5 6 7 8 9 0
```
**注意：** 数据保证一定有解。

**输入格式** :
第 1 行： 两个数，表示农场的列数 C 和行数 R。

第 2..R+1 行: 每行一个由 C 个字符组成的字符串，共同描绘出牧场地图。

**输出格式**: 一个整数，表示跳跃的最小次数。

**数据范围**
- $1≤R,C≤150$


**输入样例：**
```c
10 11
..........
....*.....
..........
...*.*....
.......*..
..*..*...H
*.........
...*...*..
.K........
...*.....*
..*....*..
```
**输出样例：**
```c
5
```

### 解法一
这马不蹩脚的（OvO）
```java
import java.util.*;
import java.io.*;

class Main {

    static int C, R;
    static int[][] w;
    static int[][] dis;
    static int[][] dir = {{-2, 1}, {1, -2}, {2, -1}, {-1, 2}, {2, 1}, {1, 2}, {-1, -2}, {-2, -1}};

    public static void main(String... args) throws Exception {
        PrintWriter out = new PrintWriter(new BufferedOutputStream(System.out));
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        // BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream("./input.txt")));
        int[] in = read(br);
        C = in[0]; R = in[1]; w = new int[R][C];
        Queue<int[]> queue = new LinkedList<>();
        dis = new int[R][C];
        for (int i = 0; i < R; i++) Arrays.fill(dis[i], -1);
        for (int i = 0; i < R; i++) {
            char[] cs = br.readLine().toCharArray();
            for (int j = 0; j < C; j++) {
                if (cs[j] == 'K') {
                    queue.add(new int[]{i, j});
                    dis[i][j] = 0;
                }
                w[i][j] = cs[j];
            }
        }
        while (!queue.isEmpty()) {
            int[] top = queue.poll();
            int x = top[0], y = top[1];
            if (w[x][y] == 'H') out.println(dis[x][y]);
            for (int i = 0; i < dir.length; i++) {
                int nx = x + dir[i][0];
                int ny = y + dir[i][1];
                if (nx < 0 || ny < 0 || nx >= R || ny >= C 
                    || dis[nx][ny] != -1 || w[nx][ny] == '*') continue;
                queue.add(new int[]{nx, ny});
                dis[nx][ny] = dis[x][y]+1;
            }
        }
        out.flush();
    }

    public static int[] read(BufferedReader br) throws Exception {
        return Arrays.stream(br.readLine().split(" ")).mapToInt(Integer::parseInt).toArray();
    }
}
```

## [1076. 迷宫问题](https://www.acwing.com/problem/content/1078/)

给定一个$n×n$的二维数组，如下所示：
```c
int maze[5][5] = {

0, 1, 0, 0, 0,

0, 1, 0, 1, 0,

0, 0, 0, 0, 0,

0, 1, 1, 1, 0,

0, 0, 0, 1, 0,

};
```
它表示一个迷宫，其中的1表示墙壁，0表示可以走的路，只能横着走或竖着走，不能斜着走，要求编程序找出从左上角到右下角的最短路线。

数据保证至少存在一条从左上角走到右下角的路径。

**输入格式**

第一行包含整数 n。接下来 n 行，每行包含 n 个整数 0 或 1，表示迷宫。

**输出格式**

输出从左上角到右下角的最短路线，如果答案不唯一，输出任意一条路径均可。按顺序，每行输出一个路径中经过的单元格的坐标，左上角坐标为$(0,0)$，右下角坐标为$(n−1,n−1)$。

**数据范围**
- $0≤n≤1000$

**输入样例：**
```c
5
0 1 0 0 0
0 1 0 1 0
0 0 0 0 0
0 1 1 1 0
0 0 0 1 0
```
**输出样例：**
```c
0 0
1 0
2 0
2 1
2 2
2 3
2 4
3 4
4 4
```

### 解法一
简单的BFS，但是要输出路径，所以我们反向搜索，然后记录每个节点的pre节点，最后再从起点倒推会更方便
```java
import java.util.*;
import java.io.*;
class Main {

    static int N;
    static int[][] w;
    static int[][] pre;
    static int[][] dir = {{0, 1}, {1, 0}, {0, -1}, {-1, 0}};

    public static void main(String []args) throws Exception {
        PrintWriter out = new PrintWriter(new BufferedOutputStream(System.out));
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        N =  read(br)[0];
        pre = new int[N][N]; 
        w = new int[N][N];
        for (int i = 0; i < N; i++) {
            w[i] = read(br);
        }
        Queue<int[]> queue = new LinkedList<>();
        boolean[][] vis = new boolean[N][N];
        //要输出路径，一般逆向搜索更加方便
        queue.add(new int[]{N-1, N-1});
        vis[N-1][N-1] = true;
        pre[N-1][N-1] = -1;
        while (!queue.isEmpty()) {
            int[] cur = queue.poll();
            if (cur[0] == 0 && cur[1] == 0) break;
            for (int i = 0; i < dir.length; i++) {
                int x = cur[0] + dir[i][0];
                int y = cur[1] + dir[i][1];
                if (x < 0 || y < 0 || x >= N || y >= N || 
                    vis[x][y] || w[x][y] == 1) continue;
                queue.add(new int[]{x, y});
                vis[x][y] = true;
                pre[x][y] = conv(cur[0], cur[1]);
            }
        }
        int tx = 0, ty = 0;
        out.println(tx + " " + ty);
        while (pre[tx][ty] != -1) {
            int[] temp = conv(pre[tx][ty]);
            out.println(temp[0] + " " + temp[1]);
            tx = temp[0]; ty = temp[1];
        }
        out.flush();
    }

    //(x,y) -> [0,N*N)
    public static int conv(int x, int y) {
        return x*N + y;
    }

    //5->(1,0)
    public static int[] conv(int i) {
        return new int[]{(i+N)/N-1, i%N};
    }

    public static int[] read(BufferedReader br) throws Exception {
        return Arrays.stream(br.readLine().split(" ")).mapToInt(Integer::parseInt).toArray();
    }
}
```

## [1100. 抓住那头牛](https://www.acwing.com/problem/content/description/1102/)

农夫知道一头牛的位置，想要抓住它。

农夫和牛都位于数轴上，农夫起始位于点$N$，牛位于点$K$。

农夫有两种移动方式：

- 从$X$移动到$X−1$或$X+1$，每次移动花费一分钟
- 从$X$移动到$2∗X$，每次移动花费一分钟

假设牛没有意识到农夫的行动，站在原地不动。

农夫最少要花多少时间才能抓住牛？

**输入格式**：共一行，包含两个整数N和K。

**输出格式**：输出一个整数，表示抓到牛所花费的最少时间。

**数据范围**: 0≤N,K≤105

**输入样例：**
```c
5 17
```
**输出样例：**
```c
4
```
### 解法一
水题，还有可以优化的地方，不过对整体时间复杂度影响不大，不做深究（py还是好写，得多用py写水题）
```python
from collections import deque

n, k = map(int, input().split())
MAX = int(1e5+10)

def solve(n, k):
    if n > k: return n-k
    q = deque()
    dis = [0 if i == n else -1 for i in range(MAX)]
    q.append(n)
    while q:
        top = q.popleft()
        if top == k:
            return dis[k]
        for ne in [top+1, top-1, top*2]:
            if ne < MAX and dis[ne] == -1:
                dis[ne] = dis[top]+1
                q.append(ne)
    return -1

print(solve(n, k))
```

# 多源BFS
## [173. 矩阵距离](https://www.acwing.com/problem/content/175/)

给定一个$N$行$M$列的$01$矩阵$A$，$A[i][j]$与$A[k][l]$之间的曼哈顿距离定义为：

$$dist(A[i][j],A[k][l])=|i−k|+|j−l|$$
输出一个$N$行$M$列的整数矩阵$B$，其中：

$$
B[i][j]=min_{1≤x≤N \And 1≤y≤M \And A[x][y]=1}(dist(A[i][j],A[x][y]))
$$

**输入格式**：第一行两个整数$N,M$。

接下来一个$N$行$M$列的$01$矩阵，数字之间没有空格。

**输出格式**：一个$N$行$M$列的矩阵$B$，相邻两个整数之间用一个空格隔开。

**数据范围**：
- $1≤N,M≤1000$

**输入样例：**
```c
3 4
0001
0011
0110
```
**输出样例：**
```c
3 2 1 0
2 1 0 0
1 0 0 1
```
### 解法一

题目意思就是把所有元素距离最近的1的曼哈顿距离算出来，正向的思考就只能一个个的去找，这样效率会比较低，所以我们完全可以逆向思考，找所有1最近的元素，先将所有的1加入队列，将其$dis$赋值为0，然后进行BFS就行了
```python
from collections import deque

n, m = map(int, input().split())
w = [list(map(int, list(input()))) for _ in range(n)]
dis = [[-1 for _ in range(m)] for _ in range(n)]

q = deque()
for i in range(n):
    for j in range(m):
        if w[i][j] == 1:
            q.append((i, j))
            dis[i][j] = 0
while q:
    x, y = q.popleft()
    for (nx, ny) in [(x + 1, y), (x, y + 1), (x - 1, y), (x, y - 1)]:
        if nx < 0 or ny < 0 or nx >= n or ny >= m or dis[nx][ny] != -1:
            continue
        dis[nx][ny] = dis[x][y] + 1
        q.append((nx, ny))

[print(' '.join([str(x) for x in r])) for r in dis]
```