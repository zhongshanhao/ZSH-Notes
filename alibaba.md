外卖配送

自己有份商品购买清单，附近的商店有这些商品，外卖配送员会帮忙配齐商品并将它送到我这里，要求在商品配齐的前题下，外卖小哥走的路线最短。

建模

用一个n*m矩阵的矩阵表示周围的世界，然后0代表可以走的路，1表示不能走的障碍物，其他字符表示商店，现在简化模型，假如一个商店只有一件商品。

现在的问题就是外卖小哥走在矩阵中，帮忙配齐商品并将其送到我家的最短路径。

问题转换

因为外卖小哥的起点是不固定的，然而我的位置是固定的，并且在所有的配送方案中，外卖小哥总是以我的位置为终点。我们可以将配送的过程反转一下，将问题转换成：从我的位置出发，去附近商店收集商品，收集完成之后的终点就是外卖小哥给我配送的起点。

```
我有个待购买的物品清单，假设我的位置在地图的左上角，规划一条路径，
使得在能够购买所有所需物品的前提下，所走的路径最短。

输入：
第一行一个数字n，代表物品清单有n件物品
第二行是代购买的物品清单
第三行有两个数字，row和col，代表地图的行数和列数
接下来有row行和col列，代表地图

3
a b c
5 6
0 0 0 a b d
0 1 1 d 0 1
0 1 b 0 0 0
c 0 a 0 c 0
0 1 0 0 1 b

ps: 1是障碍物，小红只能走上下左右四个方向

输出：我所走的最短路径的长度和及其对应的路径
```

问题解决

如果是我只要买一件物品，那么两点之间的最短距离可以很容易地使用bfs广度优先搜索来得到

- 首先我们用一个二维矩阵dp来记录源点到某一点的最短路径，dp\[x\]\[y\]就表示从源点到达x，y的最短路径

- 我们使用一个队列去模拟bfs搜索过程，首先将源点加入队列

- 当队列不为空时，循环执行以下流程
  - 将队列出队，然后判断当前位置能够购买我需要的那件物品，如果是，直接break，那么终点位置的dp值就表示从源点到达终点的最短路径长度。
  - 如果当前位置不能购买，枚举周围可以移动的并且没有被访问过的位置，将符合条件的位置加入到队列中，同时更新dp数组，将符合条件的位置上的步数更新为当前位置的步数加一。

这是找两个点之间最短路径的解法

但现在是要到达多个点，求到达多个点的最短路径。还是考虑使用二维dp数组记录的情况，显然，当一件商品在一条死胡同里的时候，就是一条路线需要被反复走的时候，使用二维的dp数组记录某个位置的点是否被访问过是不能够解决该问题的。

这样，我将获得物品的状态state加入到数组中的第三维，就获得一个三维数组，在我获得某件商品的时候更新state，那么，在这个新的state下的三维数组所有位置又都是未经访问的，此时bfs就可以原路返回。

```java
public class Main {
    static String path = "D:\\workplace\\idea-project\\HdfsClientDemo\\src\\main\\resources\\input";
    static final int[] dx = new int[]{1, 0, -1, 0};
    static final int[] dy = new int[]{0, 1, 0, -1};
    
    class Node{
        int x;
        int y;
        int state;
        int step;
        
        Node(int x, int y, int state, int step){
            this.x = x;
            this.y = y;
            this.state = state;
            this.step = step;
        }

        @Override
        public String toString() {
            return x + "_" + y +"_" + state;
        }
    }
    
    static int whichGoods(char c){ 
        int i = c - 'a';
        if(i >= 0 && i < 26)
            return i;
        return -1;
    }
    
    public List<int[]> find(char[][] graph, Set<Character> list) {
        int target = 0;
        int num = list.size();
        int row = graph.length;
        int col = graph[0].length;
        boolean[][][] vis = new boolean[row][col][1 << num];
        Deque<Node> queue = new LinkedList<>();
        Map<String, String> path = new HashMap<>();
        int endX = -1;
        int endY = -1;
        // 遍历订单列表，获得目标状态
        for(char c : list){
            int i = whichGoods(c);
            if(i != -1){
                target |= (1 << i);
            }
        }
        int originState = 0;
        int originX = 0;
        int originY = 0;
        int goods = whichGoods(graph[originX][originY]);
        if(goods != -1 && list.contains(graph[originX][originY]))
            originState |= (1 << goods);
        vis[originX][originY][originState] = true;
        String start = originX + "_" + originY + "_" + originState;
        queue.add(new Node(originX, originX, originState, 0));
        while(queue.size() > 0){
            Node k = queue.remove();
            if(k.state == target){
                endX = k.x;
                endY = k.y;
                break;
            }
            for(int i = 0; i < 4; i++){
                int x = k.x + dx[i];
                int y = k.y + dy[i];
                int newState = k.state;
                int curStep = k.step;
                if(x >= row || x < 0 || y >= col || y < 0 || graph[x][y] == '1')
                    continue;
                goods = whichGoods(graph[x][y]);
                if(goods != -1 && list.contains(graph[x][y]))
                    newState |= (1 << goods);
                if(vis[x][y][newState] == false){
                    vis[x][y][newState] = true;
                    Node node = new Node(x, y, newState,curStep + 1);
                    path.put(node.toString(), k.toString());
                    queue.add(node);
                }
            }
        }
        List<int[]> ans = new ArrayList<>();
        if(endX == -1)
            return ans;
        String s = endX + "_" + endY + "_" + target;
        // System.out.println(s);
        while(s != null && !start.equals(s)){
            // System.out.println(s);
            String[] p = s.split("_");
            ans.add(new int[]{Integer.parseInt(p[0]), Integer.parseInt(p[1])});
            s = path.get(s);
        }
        ans.add(new int[]{0, 0});
        return ans;
    }
    
    public static void main(String[] args) throws IOException {
        File file = new File(path);
        Scanner scanner = new Scanner(file);
        int num = scanner.nextInt();
        scanner.nextLine();
        String[] goods = scanner.nextLine().split(" ");
        Set<Character> list = new HashSet<>();
        for(String good : goods){
            list.add(good.charAt(0));
        }
        int row = scanner.nextInt();
        int col = scanner.nextInt();
        char[][] graph = new char[row][col];
        for(int i = 0; i < row; i++){
            for(int j = 0; j < col; j++){
                graph[i][j] = scanner.next().charAt(0);
            }
        }
        for(int i = 0; i < row; i++){
            for(int j = 0; j < col; j++){
                System.out.print(graph[i][j] + " ");
            }
            System.out.println();
        }
        
        List<int[]> path = new Main().find(graph, list);

        System.out.println("最短路径长度：" + (path.size() - 1));
        System.out.println("路线如下：");
        for(int[] p : path){
            System.out.println(p[0] + " " +p[1]);    
        }
    }
}
```

输入

```java
3
a b c
5 6
0 0 0 a b d
0 1 1 d 0 1
0 1 b 0 0 0
c 0 a 0 c 0
0 1 0 0 1 b
```

输出

```java
0 0 0 a b d 
0 1 1 d 0 1 
0 1 b 0 0 0 
c 0 a 0 c 0 
0 1 0 0 1 b 
最短路径长度：6
路线如下：
2 2
3 2
3 1
3 0
2 0
1 0
0 0
```