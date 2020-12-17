---
title: 
   搜索：Flood Fill
tags: 
  [算法]
categories:
    [算法]
---

## [1097. 池塘计数](https://www.acwing.com/problem/content/1099/)

农夫约翰有一片 N∗M 的矩形土地。

最近，由于降雨的原因，部分土地被水淹没了。

现在用一个字符矩阵来表示他的土地。

每个单元格内，如果包含雨水，则用”W”表示，如果不含雨水，则用”.”表示。

现在，约翰想知道他的土地中形成了多少片池塘。

每组相连的积水单元格集合可以看作是一片池塘。

每个单元格视为与其上、下、左、右、左上、右上、左下、右下八个邻近单元格相连。

请你输出共有多少片池塘，即矩阵中共有多少片相连的”W”块。

**输入格式**

第一行包含两个整数 N 和 M。

接下来 N 行，每行包含 M 个字符，字符为”W”或”.”，用以表示矩形土地的积水状况，字符之间没有空格。

**输出格式**

输出一个整数，表示池塘数目。

**数据范围**：1≤N,M≤1000

**输入样例：**
```c
10 12
W........WW.
.WWW.....WWW
....WW...WW.
.........WW.
.........W..
..W......W..
.W.W.....WW.
W.W.W.....W.
.W.W......W.
..W.......W.

```
**输出样例：**
```c
3
```

### 解法一

经典Flood Fill，搜索或者并查集都可，没啥好说的
```java
import java.io.*;
import java.util.*;
class Main {
    
    static int N;
    static int M;
    static boolean[][] vis;    
    static char[][] board;
    static int[][] dir = {{0, 1}, {1, 0}, {0, -1}, {-1, 0}, {1, -1}, {-1, 1}, {-1, -1}, {1, 1}};

    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        N = sc.nextInt();
        M = sc.nextInt();
        board = new char[N][M];
        vis = new boolean[N][M];
        for (int i = 0; i < N; i++) {
            board[i] = sc.next().toCharArray();
        }
        int res = 0;
        for (int i = 0; i < N; i++) {
            for (int j = 0; j < M; j++) {
                if (!vis[i][j] && board[i][j] == 'W') {
                    res++;
                    dfs(i, j);
                }
            }
        }
        System.out.println(res);
    }

    public static void dfs (int x, int y) {
        vis[x][y] = true;
        for (int i = 0; i < dir.length; i++) {
            int nx = x + dir[i][0];
            int ny = y + dir[i][1];
            if (nx < 0 || ny < 0 || nx >= N || ny >= M) {
                continue;
            }
            if (!vis[nx][ny] && board[nx][ny] == 'W') {
                dfs(nx, ny);
            }
        }
    }
}
```

## [1098. 城堡问题](https://www.acwing.com/problem/content/1100/)

```c
    1   2   3   4   5   6   7  
   #############################
 1 #   |   #   |   #   |   |   #
   #####---#####---#---#####---#
 2 #   #   |   #   #   #   #   #
   #---#####---#####---#####---#
 3 #   |   |   #   #   #   #   #
   #---#########---#####---#---#
 4 #   #   |   |   |   |   #   #
   #############################
           (图 1)
   #  = Wall   
   |  = No wall
   -  = No wall
```

   方向：上北下南左西右东。
图1是一个城堡的地形图。

请你编写一个程序，计算城堡一共有多少房间，最大的房间有多大。

城堡被分割成 m∗n个方格区域，每个方格区域可以有0~4面墙。

注意：墙体厚度忽略不计。

输入格式
第一行包含两个整数 m 和 n，分别表示城堡南北方向的长度和东西方向的长度。

接下来 m 行，每行包含 n 个整数，每个整数都表示平面图对应位置的方块的墙的特征。

每个方块中墙的特征由数字 P 来描述，我们用1表示西墙，2表示北墙，4表示东墙，8表示南墙，P 为该方块包含墙的数字之和。

例如，如果一个方块的 P 为3，则 3 = 1 + 2，该方块包含西墙和北墙。

城堡的内墙被计算两次，方块(1,1)的南墙同时也是方块(2,1)的北墙。

输入的数据保证城堡至少有两个房间。

**输出格式**

共两行，第一行输出房间总数，第二行输出最大房间的面积（方块数）。

**数据范围**：1≤m,n≤50, 0≤P≤15

**输入样例：**
```c
4 7 
11 6 11 6 3 10 6 
7 9 6 13 5 15 5 
1 10 12 7 13 7 5 
13 11 10 8 10 12 13 
```
**输出样例：**
```c
5
9
```

### 解法一

题目很简单，但是怎么处理墙有点麻烦，我直接采用了最蠢的办法

```java
import java.io.*;
import java.util.*;

public class AcWing1098_城堡问题 {
    public static void main(String[] args) throws Exception {
        new Main().main();
    }
}

class Main {
    //     2(3)
    //1(2)      4(1)
    //     8(0)
    
    //下，右，左，上
    static int[][] dir = {{1, 0}, {0, 1}, {0, -1}, {-1, 0}};    
    static int n, m;
    static boolean[][] wall;
    static boolean[][] vis;
    static int[][] board;
    
    public static void main(String... args) throws Exception {
        Scanner sc = new Scanner(new File("./input.txt"));
        // Scanner sc = new Scanner(System.in);
        n = sc.nextInt();
        m = sc.nextInt();
        vis = new boolean[n][m];
        wall = new boolean[16][4];
        wall[2][3] = wall[1][2] = wall[4][1] = wall[8][0] = true;
        wall[3][2] = wall[3][3] = true;
        wall[5][1] = wall[5][2] = true;
        wall[9][0] = wall[9][2] = true;
        wall[6][1] = wall[6][3] = true;
        wall[10][0] = wall[10][3] = true;
        wall[12][0] = wall[12][1] = true;
        wall[7][1] = wall[7][2] = wall[7][3] = true; 
        wall[11][0] = wall[11][2] = wall[11][3] = true;
        wall[14][0] = wall[14][1] = wall[14][3] = true;
        wall[13][0] = wall[13][1] = wall[13][2] = true;
        wall[15][0] = wall[15][1] = wall[15][2] = wall[15][3] = true;
        board = new int[n][m];
        for (int i = 0; i < n; i++){
            for (int j = 0; j < m; j++) {
                board[i][j] = sc.nextInt();
            }
        }
        int count = 0, max = 0;
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                if (!vis[i][j]) {
                    count++;
                    max = Math.max(max, dfs(i, j));
                }
            }
        }
        System.out.println(count);
        System.out.println(max);
    }

    public static int dfs (int x, int y) {
        vis[x][y] = true;
        int area = 1;
        for (int i = 0; i < dir.length; i++) {
            int nx = x + dir[i][0];
            int ny = y + dir[i][1];
            if (nx < 0 || ny < 0 || nx >= n || ny >= m || wall[board[x][y]][i]) {
                continue;
            }
            if (!vis[nx][ny]) {
                area += dfs(nx, ny);
            }
        }
        return area;
    }
}
```
### 解法二

优雅的位运算，东南西北对应的墙1，2，4，8就是2<sup>0</sup>，2<sup>1</sup>，2<sup>2</sup>，2<sup>3</sup>，所以我们可以根据二进制的对应位的0，1状态判断该方向有没有墙

```java
class Main {
    //        2(0010)
    //1(0001)         4(0100)
    //        8(1000)
    //左，上，右，下
    static int[][] dir = {{0, -1}, {-1, 0}, {0, 1}, {1, 0}};    
    static int n, m;
    static boolean[][] vis;
    static int[][] board;
    
    public static void main(String... args) throws Exception {
        Scanner sc = new Scanner(new File("./input.txt"));
        // Scanner sc = new Scanner(System.in);
        n = sc.nextInt();
        m = sc.nextInt();
        vis = new boolean[n][m];
        board = new int[n][m];
        for (int i = 0; i < n; i++){
            for (int j = 0; j < m; j++) {
                board[i][j] = sc.nextInt();
            }
        }
        int count = 0, max = 0;
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                if (!vis[i][j]) {
                    count++;
                    max = Math.max(max, dfs(i, j));
                }
            }
        }
        System.out.println(count);
        System.out.println(max);
    }

    public static int dfs (int x, int y) {
        vis[x][y] = true;
        int area = 1;
        for (int i = 0; i < dir.length; i++) {
            int nx = x + dir[i][0];
            int ny = y + dir[i][1];
            if (nx < 0 || ny < 0 || nx >= n || ny >= m) {
                continue;
            }
            if (!vis[nx][ny] && ((board[x][y]>>i)&1) == 0) {
                area += dfs(nx, ny);
            }
        }
        return area;
    }
}
```