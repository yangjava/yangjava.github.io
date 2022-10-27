


















### 桶

另外一种方法类似于链地址法，它是在每个数据项中使用子数组，而不是链表。这样的数组称为桶。

　　这个方法显然不如链表有效，因为桶的容量不好选择，如果容量太小，可能会溢出，如果太大，又造成性能浪费，而链表是动态分配的，不存在此问题。所以一般不使用桶。

哈希表基于数组，类似于key-value的存储形式，关键字值通过哈希函数映射为数组的下标，如果一个关键字哈希化到已占用的数组单元，这种情况称为冲突。用来解决冲突的有两种方法：开放地址法和链地址法。在开发地址法中，把冲突的数据项放在数组的其它位置；在链地址法中，每个单元都包含一个链表，把所有映射到同一数组下标的数据项都插入到这个链表中。

## 堆

优先级队列是一种抽象数据类型（ADT），它提供了删除最大（或最小）关键字值的数据项的方法，插入数据项的方法，优先级队列可以用有序数组来实现，这种实现方式尽管删除最大数据项的时间复杂度为O(1)，但是插入还是需要较长的时间 O(N)，因为每次插入平均需要移动一半的数据项，来保证插入后，数组依旧有序。

　　本篇博客我们介绍另外一种数据结构——堆，注意这里的堆和我们Java语言，C++语言等编程语言在内存中的“堆”是不一样的，这里的堆是一种树，由它实现的优先级队列的插入和删除的时间复杂度都为O(logN)，这样尽管删除的时间变慢了，但是插入的时间快了很多，当速度非常重要，而且有很多插入操作时，可以选择用堆来实现优先级队列。

### 堆的定义

　①、它是完全二叉树，除了树的最后一层节点不需要是满的，其它的每一层从左到右都是满的。注意下面两种情况，第二种最后一层从左到右中间有断隔，那么也是不完全二叉树。

　　②、它通常用数组来实现。

　　这种用数组实现的二叉树，假设节点的索引值为index，那么：

　　节点的左子节点是 2*index+1，

　　节点的右子节点是 2*index+2，

　　节点的父节点是 （index-1）/2。

　　③、堆中的每一个节点的关键字都大于（或等于）这个节点的子节点的关键字。

　　这里要注意堆和前面说的二叉搜索树的区别，二叉搜索树中所有节点的左子节点关键字都小于右子节点关键字，在二叉搜索树中通过一个简单的算法就可以按序遍历节点。但是在堆中，按序遍历节点是很困难的，如上图所示，堆只有沿着从根节点到叶子节点的每一条路径是降序排列的，指定节点的左边节点或者右边节点，以及上层节点或者下层节点由于不在同一条路径上，他们的关键字可能比指定节点大或者小。所以相对于二叉搜索树，堆是弱序的。

### 遍历和查找

前面我们说了，堆是弱序的，所以想要遍历堆是很困难的，基本上，堆是不支持遍历的。

　　对于查找，由于堆的特性，在查找的过程中，没有足够的信息来决定选择通过节点的两个子节点中的哪一个来选择走向下一层，所以也很难在堆中查找到某个关键字。

　　因此，堆这种组织似乎非常接近无序，不过，对于快速的移除最大（或最小）节点，也就是根节点，以及能快速插入新的节点，这两个操作就足够了。

### 移除

　　移除是指删除关键字最大的节点（或最小），也就是根节点。

　　根节点在数组中的索引总是0，即maxNode = heapArray[0];

　　移除根节点之后，那树就空了一个根节点，也就是数组有了一个空的数据单元，这个空单元我们必须填上。

　　第一种方法：将数组所有数据项都向前移动一个单元，这比较费时。

　　第二种方法：

　　　　①、移走根

　　　　②、把最后一个节点移动到根的位置

　　　　③、一直向下筛选这个节点，直到它在一个大于它的节点之下，小于它的节点之上为止。

　　具体步骤如下：

　　图a表示把最后一个节点移到根节点，图b、c、d表示将节点向下筛选到合适的位置，它的合适位置在最底层（有时候可能在中间），图e表示节点在正确位置的情景。

　　注意：向下筛选的时候，将目标节点和其子节点比较，谁大就和谁交换位置。

### 插入

　插入节点也很容易，插入时，选择向上筛选，节点初始时插入到数组最后第一个空着的单元，数组容量大小增一。然后进行向上筛选的算法。

　　注意：向上筛选和向下不同，向上筛选只用和一个父节点进行比较，比父节点小就停止筛选了。

　

### 完整的Java堆代码

　　首先我们要知道用数组表示堆的一些要点。若数组中节点的索引为x，则：

　　节点的左子节点是 2*index+1，

　　节点的右子节点是 2*index+2，

　　节点的父节点是 （index-1）/2。

　　注意："/" 这个符号，应用于整数的算式时，它执行整除，且得到是是向下取整的值。

```
 
public class Heap {
     
    private Node[] heapArray;
    private int maxSize;
    private int currentSize;
     
    public Heap(int mx) {
        maxSize = mx;
        currentSize = 0;
        heapArray = new Node[maxSize];
    }
     
    public boolean isEmpty() {
        return (currentSize == 0)? true : false;
    }
     
    public boolean isFull() {
        return (currentSize == maxSize)? true : false;
    }
     
    public boolean insert(int key) {
        if(isFull()) {
            return false;
        }
        Node newNode = new Node(key);
        heapArray[currentSize] = newNode;
        trickleUp(currentSize++);
        return true;
    }
    //向上调整
    public void trickleUp(int index) {
        int parent = (index - 1) / 2; //父节点的索引
        Node bottom = heapArray[index]; //将新加的尾节点存在bottom中
        while(index > 0 && heapArray[parent].getKey() < bottom.getKey()) {
            heapArray[index] = heapArray[parent];
            index = parent;
            parent = (parent - 1) / 2;
        }
        heapArray[index] = bottom;
    }
     
    public Node remove() {
        Node root = heapArray[0];
        heapArray[0] = heapArray[--currentSize];
        trickleDown(0);
        return root;
    }
    //向下调整
    public void trickleDown(int index) {
        Node top = heapArray[index];
        int largeChildIndex;
        while(index < currentSize/2) { //while node has at least one child
            int leftChildIndex = 2 * index + 1;
            int rightChildIndex = leftChildIndex + 1;
            //find larger child
            if(rightChildIndex < currentSize &&  //rightChild exists?
                    heapArray[leftChildIndex].getKey() < heapArray[rightChildIndex].getKey()) {
                largeChildIndex = rightChildIndex;
            }
            else {
                largeChildIndex = leftChildIndex;
            }
            if(top.getKey() >= heapArray[largeChildIndex].getKey()) {
                break;
            }
            heapArray[index] = heapArray[largeChildIndex];
            index = largeChildIndex;
        }
        heapArray[index] = top;
    }
    //根据索引改变堆中某个数据
    public boolean change(int index, int newValue) {
        if(index < 0 || index >= currentSize) {
            return false;
        }
        int oldValue = heapArray[index].getKey();
        heapArray[index].setKey(newValue);
        if(oldValue < newValue) {
            trickleUp(index);
        }
        else {
            trickleDown(index);
        }
        return true;
    }
     
    public void displayHeap() {
        System.out.println("heapArray(array format): ");
        for(int i = 0; i < currentSize; i++) {
            if(heapArray[i] != null) {
                System.out.print(heapArray[i].getKey() + " ");
            }
            else {
                System.out.print("--");
            }
        }
    }
}
class Node {
    private int iData;
    public Node(int key) {
        iData = key;
    }
     
    public int getKey() {
        return iData;
    }
     
    public void setKey(int key) {
        iData = key;
    }
}
```

## 图(Graph)

图也是计算机程序设计中最常用的数据结构之一，从数学意义上讲，树是图的一种，大家可以对比着学习。

### 图的定义

![数据结构-graph](png\Java\数据结构-graph.png)

**图(graph)**由多个**节点(vertex)**构成，节点之间阔以互相连接组成一个网络。(x, y)表示一条**边(edge)**，它表示节点 x 与 y 相连。边可能会有**权值(weight/cost)**。

#### 邻接：

　　如果两个顶点被同一条边连接，就称这两个顶点是邻接的，如上图 I 和 G 就是邻接的，而 I 和 F 就不是。有时候也将和某个指定顶点邻接的顶点叫做它的邻居，比如顶点 G 的邻居是 I、H、F。

#### 路径：

　　路径是边的序列，比如从顶点B到顶点J的路径为 BAEJ，当然还有别的路径 BCDJ，BACDJ等等。

#### 连通图和非连通图：

　　如果至少有一条路径可以连接起所有的顶点，那么这个图称作连通的；如果假如存在从某个顶点不能到达另外一个顶点，则称为非联通的。

#### 有向图和无向图：

　　如果图中的边没有方向，可以从任意一边到达另一边，则称为无向图；比如双向高速公路，A城市到B城市可以开车从A驶向B，也可以开车从B城市驶向A城市。但是如果只能从A城市驶向B城市的图，那么则称为有向图。

#### 有权图和无权图：

　　图中的边被赋予一个权值，权值是一个数字，它能代表两个顶点间的物理距离，或者从一个顶点到另一个顶点的时间，这种图被称为有权图；反之边没有赋值的则称为无权图。

　　本篇博客我们讨论的是无权无向图。

### 在程序中表示图

　我们知道图是由顶点和边组成，那么在计算机中，怎么来模拟顶点和边？

#### 顶点：

　　在大多数情况下，顶点表示某个真实世界的对象，这个对象必须用数据项来描述。比如在一个飞机航线模拟程序中，顶点表示城市，那么它需要存储城市的名字、海拔高度、地理位置和其它相关信息，因此通常用一个顶点类的对象来表示一个顶点，这里我们仅仅在顶点中存储了一个字母来标识顶点，同时还有一个标志位，用来判断该顶点有没有被访问过（用于后面的搜索）。

```
/**
 * 顶点类
 * @author vae
 */
public class Vertex {
    public char label;
    public boolean wasVisited;
     
    public Vertex(char label){
        this.label = label;
        wasVisited = false;
    }
}
```

　　顶点对象能放在数组中，然后用下标指示，也可以放在链表或其它数据结构中，不论使用什么结构，存储只是为了使用方便，这与边如何连接点是没有关系的。

#### 边：

　　在前面讲解各种树的数据结构时，大多数树都是每个节点包含它的子节点的引用，比如红黑树、二叉树。也有用数组表示树，树组中节点的位置决定了它和其它节点的关系，比如堆就是用数组表示。

　　然而图并不像树，图没有固定的结构，图的每个顶点可以与任意多个顶点相连，为了模拟这种自由形式的组织结构，用如下两种方式表示图：邻接矩阵和邻接表（如果一条边连接两个顶点，那么这两个顶点就是邻接的）

　　**邻接矩阵：**

　　邻接矩阵是一个二维数组，数据项表示两点间是否存在边，如果图中有 N 个顶点，邻接矩阵就是 N*N 的数组。上图用邻接矩阵表示如下：

　　![img](https://images2017.cnblogs.com/blog/1120165/201802/1120165-20180209221722904-1549384558.png)

　　1表示有边，0表示没有边，也可以用布尔变量true和false来表示。顶点与自身相连用 0 表示，所以这个矩阵从左上角到右上角的对角线全是 0 。

　　注意：这个矩阵的上三角是下三角的镜像，两个三角包含了相同的信息，这个冗余信息看似低效，但是在大多数计算机中，创造一个三角形数组比较困难，所以只好接受这个冗余，这也要求在程序处理中，当我们增加一条边时，比如更新邻接矩阵的两部分，而不是一部分。

　　**邻接表：**

　　邻接表是一个链表数组（或者是链表的链表），每个单独的链表表示了有哪些顶点与当前顶点邻接。

　　![img](https://images2017.cnblogs.com/blog/1120165/201802/1120165-20180209222328123-1963427649.png)　　

### 搜索

　　在图中实现最基本的操作之一就是搜索从一个指定顶点可以到达哪些顶点，比如从武汉出发的高铁可以到达哪些城市，一些城市可以直达，一些城市不能直达。现在有一份全国高铁模拟图，要从某个城市（顶点）开始，沿着铁轨（边）移动到其他城市（顶点），有两种方法可以用来搜索图：深度优先搜索（DFS）和广度优先搜索（BFS）。它们最终都会到达所有连通的顶点，深度优先搜索通过栈来实现，而广度优先搜索通过队列来实现，不同的实现机制导致不同的搜索方式。

#### 深度优先搜索（DFS）

　　深度优先搜索算法有如下规则：

　　规则1：如果可能，访问一个邻接的未访问顶点，标记它，并将它放入栈中。

　　规则2：当不能执行规则 1 时，如果栈不为空，就从栈中弹出一个顶点。

　　规则3：如果不能执行规则 1 和规则 2 时，就完成了整个搜索过程。

　　![img](https://images2017.cnblogs.com/blog/1120165/201802/1120165-20180209225344107-1667286888.png)

　　对于上图，应用深度优先搜索如下：假设选取 A 顶点为起始点，并且按照字母优先顺序进行访问，那么应用规则 1 ，接下来访问顶点 B，然后标记它，并将它放入栈中；再次应用规则 1，接下来访问顶点 F，再次应用规则 1，访问顶点 H。我们这时候发现，没有 H 顶点的邻接点了，这时候应用规则 2，从栈中弹出 H，这时候回到了顶点 F，但是我们发现 F 也除了 H 也没有与之邻接且未访问的顶点了，那么再弹出 F，这时候回到顶点 B，同理规则 1 应用不了，应用规则 2，弹出 B，这时候栈中只有顶点 A了，然后 A 还有未访问的邻接点，所有接下来访问顶点 C，但是 C又是这条线的终点，所以从栈中弹出它，再次回到 A，接着访问 D,G,I，最后也回到了 A，然后访问 E，但是最后又回到了顶点 A，这时候我们发现 A没有未访问的邻接点了，所以也把它弹出栈。现在栈中已无顶点，于是应用规则 3，完成了整个搜索过程。

　　深度优先搜索在于能够找到与某一顶点邻接且没有访问过的顶点。这里以邻接矩阵为例，找到顶点所在的行，从第一列开始向后寻找值为1的列；列号是邻接顶点的号码，检查这个顶点是否未访问过，如果是这样，那么这就是要访问的下一个顶点，如果该行没有顶点既等于1（邻接）且又是未访问的，那么与指定点相邻接的顶点就全部访问过了（后面会用算法实现）。

#### 广度优先搜索（BFS）

　　深度优先搜索要尽可能的远离起始点，而广度优先搜索则要尽可能的靠近起始点，它首先访问起始顶点的所有邻接点，然后再访问较远的区域，这种搜索不能用栈实现，而是用队列实现。

　　规则1：访问下一个未访问的邻接点（如果存在），这个顶点必须是当前顶点的邻接点，标记它，并把它插入到队列中。

　　规则2：如果已经没有未访问的邻接点而不能执行规则 1 时，那么从队列列头取出一个顶点（如果存在），并使其成为当前顶点。

　　规则3：如果因为队列为空而不能执行规则 2，则搜索结束。

　　对于上面的图，应用广度优先搜索：以A为起始点，首先访问所有与 A 相邻的顶点，并在访问的同时将其插入队列中，现在已经访问了 A,B,C,D和E。这时队列（从头到尾）包含 BCDE，已经没有未访问的且与顶点 A 邻接的顶点了，所以从队列中取出B，寻找与B邻接的顶点，这时找到F，所以把F插入到队列中。已经没有未访问且与B邻接的顶点了，所以从队列列头取出C，它没有未访问的邻接点。因此取出 D 并访问 G，D也没有未访问的邻接点了，所以取出E，现在队列中有 FG，在取出 F，访问 H，然后取出 G，访问 I，现在队列中有 HI，当取出他们时，发现没有其它为访问的顶点了，这时队列为空，搜索结束。

#### 程序实现

实现深度优先搜索的栈 StackX.class

```
 
public class StackX {
    private final int SIZE = 20;
    private int[] st;
    private int top;
     
    public StackX(){
        st = new int[SIZE];
        top = -1;
    }
     
    public void push(int j){
        st[++top] = j;
    }
     
    public int pop(){
        return st[top--];
    }
     
    public int peek(){
        return st[top];
    }
     
    public boolean isEmpty(){
        return (top == -1);
    }
 
}
```

实现广度优先搜索的队列Queue.class

```
 
public class Queue {
    private final int SIZE = 20;
    private int[] queArray;
    private int front;
    private int rear;
     
    public Queue(){
        queArray = new int[SIZE];
        front = 0;
        rear = -1;
    }
     
    public void insert(int j) {
        if(rear == SIZE-1) {
            rear = -1;
        }
        queArray[++rear] = j;
    }
     
    public int remove() {
        int temp = queArray[front++];
        if(front == SIZE) {
            front = 0;
        }
        return temp;
    }
     
    public boolean isEmpty() {
        return (rear+1 == front || front+SIZE-1 == rear);
    }
}
```

图代码 Graph.class

```
 
public class Graph {
    private final int MAX_VERTS = 20;//表示顶点的个数
    private Vertex vertexList[];//用来存储顶点的数组
    private int adjMat[][];//用邻接矩阵来存储 边,数组元素0表示没有边界，1表示有边界
    private int nVerts;//顶点个数
    private StackX theStack;//用栈实现深度优先搜索
    private Queue queue;//用队列实现广度优先搜索
    /**
     * 顶点类
     * @author vae
     */
    class Vertex {
        public char label;
        public boolean wasVisited;
         
        public Vertex(char label){
            this.label = label;
            wasVisited = false;
        }
    }
     
    public Graph(){
        vertexList = new Vertex[MAX_VERTS];
        adjMat = new int[MAX_VERTS][MAX_VERTS];
        nVerts = 0;//初始化顶点个数为0
        //初始化邻接矩阵所有元素都为0，即所有顶点都没有边
        for(int i = 0; i < MAX_VERTS; i++) {
            for(int j = 0; j < MAX_VERTS; j++) {
                adjMat[i][j] = 0;
            }
        }
        theStack = new StackX();
        queue = new Queue();
    }
     
    //将顶点添加到数组中，是否访问标志置为wasVisited=false(未访问)
    public void addVertex(char lab) {
        vertexList[nVerts++] = new Vertex(lab);
    }
     
    //注意用邻接矩阵表示边，是对称的，两部分都要赋值
    public void addEdge(int start, int end) {
        adjMat[start][end] = 1;
        adjMat[end][start] = 1;
    }
     
    //打印某个顶点表示的值
    public void displayVertex(int v) {
        System.out.print(vertexList[v].label);
    }
    /**深度优先搜索算法:
     * 1、用peek()方法检查栈顶的顶点
     * 2、用getAdjUnvisitedVertex()方法找到当前栈顶点邻接且未被访问的顶点
     * 3、第二步方法返回值不等于-1则找到下一个未访问的邻接顶点，访问这个顶点，并入栈
     *    如果第二步方法返回值等于 -1，则没有找到，出栈
     */
    public void depthFirstSearch() {
        //从第一个顶点开始访问
        vertexList[0].wasVisited = true; //访问之后标记为true
        displayVertex(0);//打印访问的第一个顶点
        theStack.push(0);//将第一个顶点放入栈中
         
        while(!theStack.isEmpty()) {
            //找到栈当前顶点邻接且未被访问的顶点
            int v = getAdjUnvisitedVertex(theStack.peek());
            if(v == -1) {   //如果当前顶点值为-1，则表示没有邻接且未被访问顶点，那么出栈顶点
                theStack.pop();
            }else { //否则访问下一个邻接顶点
                vertexList[v].wasVisited = true;
                displayVertex(v);
                theStack.push(v);
            }
        }
         
        //栈访问完毕，重置所有标记位wasVisited=false
        for(int i = 0; i < nVerts; i++) {
            vertexList[i].wasVisited = false;
        }
    }
     
    //找到与某一顶点邻接且未被访问的顶点
    public int getAdjUnvisitedVertex(int v) {
        for(int i = 0; i < nVerts; i++) {
            //v顶点与i顶点相邻（邻接矩阵值为1）且未被访问 wasVisited==false
            if(adjMat[v][i] == 1 && vertexList[i].wasVisited == false) {
                return i;
            }
        }
        return -1;
    }
     
    /**
     * 广度优先搜索算法：
     * 1、用remove()方法检查栈顶的顶点
     * 2、试图找到这个顶点还未访问的邻节点
     * 3、 如果没有找到，该顶点出列
     * 4、 如果找到这样的顶点，访问这个顶点，并把它放入队列中
     */
    public void breadthFirstSearch(){
        vertexList[0].wasVisited = true;
        displayVertex(0);
        queue.insert(0);
        int v2;
         
        while(!queue.isEmpty()) {
            int v1 = queue.remove();
            while((v2 = getAdjUnvisitedVertex(v1)) != -1) {
                vertexList[v2].wasVisited = true;
                displayVertex(v2);
                queue.insert(v2);
            }
        }
         
        //搜索完毕，初始化，以便于下次搜索
        for(int i = 0; i < nVerts; i++) {
            vertexList[i].wasVisited = false;
        }
    }
     
    public static void main(String[] args) {
        Graph graph = new Graph();
        graph.addVertex('A');
        graph.addVertex('B');
        graph.addVertex('C');
        graph.addVertex('D');
        graph.addVertex('E');
         
        graph.addEdge(0, 1);//AB
        graph.addEdge(1, 2);//BC
        graph.addEdge(0, 3);//AD
        graph.addEdge(3, 4);//DE
         
        System.out.println("深度优先搜索算法 :");
        graph.depthFirstSearch();//ABCDE
         
        System.out.println();
        System.out.println("----------------------");
         
        System.out.println("广度优先搜索算法 :");
        graph.breadthFirstSearch();//ABDCE
    }
}
```

#### 最小生成树

　对于图的操作，还有一个最常用的就是找到最小生成树，最小生成树就是用最少的边连接所有顶点。对于给定的一组顶点，可能有很多种最小生成树，但是最小生成树的边的数量 E 总是比顶点 V 的数量小1，即：

　　V = E + 1

　　这里不用关心边的长度，不是找最短的路径（会在带权图中讲解），而是找最少数量的边，可以基于深度优先搜索和广度优先搜索来实现。

　　比如基于深度优先搜索，我们记录走过的边，就可以创建一个最小生成树。因为DFS 访问所有顶点，但只访问一次，它绝对不会两次访问同一个顶点，但她看到某条边将到达一个已访问的顶点，它就不会走这条边，它从来不遍历那些不可能的边，因此，DFS 算法走过整个图的路径必定是最小生成树。

```
//基于深度优先搜索找到最小生成树
public void mst(){
    vertexList[0].wasVisited = true;
    theStack.push(0);
     
    while(!theStack.isEmpty()){
        int currentVertex = theStack.peek();
        int v = getAdjUnvisitedVertex(currentVertex);
        if(v == -1){
            theStack.pop();
        }else{
            vertexList[v].wasVisited = true;
            theStack.push(v);
             
            displayVertex(currentVertex);
            displayVertex(v);
            System.out.print(" ");
        }
    }
     
    //搜索完毕，初始化，以便于下次搜索
    for(int i = 0; i < nVerts; i++) {
        vertexList[i].wasVisited = false;
    }
}
```

图是由边连接的顶点组成，图可以表示许多真实的世界情况，包括飞机航线、电子线路等等。搜索算法以一种系统的方式访问图中的每个顶点，主要通过深度优先搜索（DFS）和广度优先搜索（BFS），深度优先搜索通过栈来实现，广度优先搜索通过队列来实现。最后需要知道最小生成树是包含连接图中所有顶点所需要的最少数量的边。

## 前缀树（Trie）

假如有一个字典，字典里面有如下词："A"，"to"，"tea"，"ted"，"ten"，"i"，"in"，"inn"，每个单词还能有自己的一些权重值，那么用前缀树来构建这个字典将会是如下的样子：

![前缀树](F:/lemon-guide-main/images/Algorithm/前缀树.png)

**性质**

- 每个节点至少包含两个基本属性

  - children：数组或者集合，罗列出每个分支当中包含的所有字符
  - isEnd：布尔值，表示该节点是否为某字符串的结尾

- 前缀树的根节点是空的

  所谓空，即只利用到这个节点的 children 属性，即只关心在这个字典里，有哪些打头的字符

- 除了根节点，其他所有节点都有可能是单词的结尾，叶子节点一定都是单词的结尾



**实现**

- 创建

  - 遍历一遍输入的字符串，对每个字符串的字符进行遍历
  - 从前缀树的根节点开始，将每个字符加入到节点的 children 字符集当中
  - 如果字符集已经包含了这个字符，则跳过
  - 如果当前字符是字符串的最后一个，则把当前节点的 isEnd 标记为真。

  由上，创建的方法很直观。前缀树真正强大的地方在于，每个节点还能用来保存额外的信息，比如可以用来记录拥有相同前缀的所有字符串。因此，当用户输入某个前缀时，就能在 O(1) 的时间内给出对应的推荐字符串。

- 搜索

  与创建方法类似，从前缀树的根节点出发，逐个匹配输入的前缀字符，如果遇到了就继续往下一层搜索，如果没遇到，就立即返回。

## 线段树（Segment Tree）

假设有一个数组 array[0 … n-1]， 里面有 n 个元素，现在要经常对这个数组做两件事。

- 更新数组元素的数值
- 求数组任意一段区间里元素的总和（或者平均值）



**解法 1：遍历一遍数组**

- 时间复杂度 O(n)。

**解法 2：线段树**

- 线段树，就是一种按照二叉树的形式存储数据的结构，每个节点保存的都是数组里某一段的总和。
- 适用于数据很多，而且需要频繁更新并求和的操作。
- 时间复杂度 O(logn)。



**实现**
如数组是 [1, 3, 5, 7, 9, 11]，那么它的线段树如下。

![线段树](F:/lemon-guide-main/images/Algorithm/线段树.png)

根节点保存的是从下标 0 到下标 5 的所有元素的总和，即 36。左右两个子节点分别保存左右两半元素的总和。按照这样的逻辑不断地切分下去，最终的叶子节点保存的就是每个元素的数值。



## 树状数组（Fenwick Tree）

树状数组（Fenwick Tree / Binary Indexed Tree）。

**举例**：假设有一个数组 array[0 … n-1]， 里面有 n 个元素，现在要经常对这个数组做两件事。

- 更新数组元素的数值
- 求数组前 k 个元素的总和（或者平均值）

 

**解法 1：线段树**

- 线段树能在 O(logn) 的时间里更新和求解前 k 个元素的总和

**解法 2：树状数**

- 该问题只要求求解前 k 个元素的总和，并不要求任意一个区间
- 树状数组可以在 O(logn) 的时间里完成上述的操作
- 相对于线段树的实现，树状数组显得更简单



**树状数组特点**

- 它是利用数组来表示多叉树的结构，在这一点上和优先队列有些类似，只不过，优先队列是用数组来表示完全二叉树，而树状数组是多叉树。
- 树状数组的第一个元素是空节点。
- 如果节点 tree[y] 是 tree[x] 的父节点，那么需要满足条件：y = x - (x & (-x))。









## 算法模板

### 递归模板

```java
public void recur(int level, int param) {
    // terminator 
    if (level > MAX_LEVEL) {
        // process result    
        return;
    }

    // process current logic  
    process(level, param);

    // drill down   
    recur(level + 1, newParam);

    // restore current status 
}
```

**List转树形结构**

- **方案一：两层循环实现建树**
- **方案二：使用递归方法建树**

```java
public class TreeNode {
    private String id;
    private String parentId;
    private String name;
    private List<TreeNode> children;

    /**
     * 方案一：两层循环实现建树
     *
     * @param treeNodes 传入的树节点列表
     * @return
     */
    public static List<TreeNode> bulid(List<TreeNode> treeNodes) {
        List<TreeNode> trees = new ArrayList<>();
        for (TreeNode treeNode : treeNodes) {
            if ("0".equals(treeNode.getParentId())) {
                trees.add(treeNode);
            }

            for (TreeNode it : treeNodes) {
                if (it.getParentId() == treeNode.getId()) {
                    if (treeNode.getChildren() == null) {
                        treeNode.setChildren(new ArrayList<TreeNode>());
                    }
                    treeNode.getChildren().add(it);
                }
            }
        }

        return trees;
    }

    /**
     * 方案二：使用递归方法建树
     *
     * @param treeNodes
     * @return
     */
    public static List<TreeNode> buildByRecursive(List<TreeNode> treeNodes) {
        List<TreeNode> trees = new ArrayList<>();
        for (TreeNode treeNode : treeNodes) {
            if ("0".equals(treeNode.getParentId())) {
                trees.add(findChildren(treeNode, treeNodes));
            }
        }

        return trees;
    }

    /**
     * 递归查找子节点
     *
     * @param treeNodes
     * @return
     */
    private static TreeNode findChildren(TreeNode treeNode, List<TreeNode> treeNodes) {
        for (TreeNode it : treeNodes) {
            if (treeNode.getId().equals(it.getParentId())) {
                if (treeNode.getChildren() == null) {
                    treeNode.setChildren(new ArrayList<>());
                }
                treeNode.getChildren().add(findChildren(it, treeNodes));
            }
        }

        return treeNode;
    }
}
```



### 回溯模板



### DFS模板



### BFS模板



## 查找算法

### 顺序查找

就是一个一个依次查找。



### 二分查找

二分查找又叫折半查找，从有序列表的初始候选区`li[0:n]`开始，通过对待查找的值与候选区中间值的比较，可以使候选区减少一半。如果待查值小于候选区中间值，则只需比较中间值左边的元素，减半查找范围。依次类推依次减半。

- 二分查找的前提：**列表有序**
- 二分查找的有点：**查找速度快**
- 二分查找的时间复杂度为：**O(logn)**

![二分查找](png/Java/算法-二分查找.gif)

JAVA代码如下：

```java
/**
 * 执行递归二分查找，返回第一次出现该值的位置
 *
 * @param array     已排序的数组
 * @param start     开始位置，如：0
 * @param end       结束位置，如：array.length-1
 * @param findValue 需要找的值
 * @return 值在数组中的位置，从0开始。找不到返回-1
 */
public static int searchRecursive(int[] array, int start, int end, int findValue) {
        // 如果数组为空，直接返回-1，即查找失败
        if (array == null) {
            return -1;
        }
  
        if (start <= end) {
            // 中间位置
            int middle = (start + end) / 1;
            // 中值
            int middleValue = array[middle];
            if (findValue == middleValue) {
                // 等于中值直接返回
                return middle;
            } else if (findValue < middleValue) {
                // 小于中值时在中值前面找
                return searchRecursive(array, start, middle - 1, findValue);
            } else {
                // 大于中值在中值后面找
                return searchRecursive(array, middle + 1, end, findValue);
            }
        } else {
            // 返回-1，即查找失败
            return -1;
        }
}

/**
 * 循环二分查找，返回第一次出现该值的位置
 *
 * @param array     已排序的数组
 * @param findValue 需要找的值
 * @return 值在数组中的位置，从0开始。找不到返回-1
 */
public static int searchLoop(int[] array, int findValue) {
        // 如果数组为空，直接返回-1，即查找失败
        if (array == null) {
            return -1;
        }

        // 起始位置
        int start = 0;
        // 结束位置
        int end = array.length - 1;
        while (start <= end) {
            // 中间位置
            int middle = (start + end) / 2;
            // 中值
            int middleValue = array[middle];
            if (findValue == middleValue) {
                // 等于中值直接返回
                return middle;
            } else if (findValue < middleValue) {
                // 小于中值时在中值前面找
                end = middle - 1;
            } else {
                // 大于中值在中值后面找
                start = middle + 1;
            }
        }

        // 返回-1，即查找失败
        return -1;
}
```



### 插值查找

插值查找算法类似于二分查找，不同的是插值查找每次从自适应mid处开始查找。将折半查找中的求mid索引的公式，low表示左边索引left，high表示右边索引right，key就是前面我们讲的findVal。

![二分查找mid](png/Java/算法-二分查找mid.png)      **改为**   ![插值查找mid](F:/lemon-guide-main/images/Algorithm/插值查找mid.png)  

**注意事项**

- 对于数据量较大，关键字分布比较均匀的查找表来说，采用插值查找，速度较快
- 关键字分布不均匀的情况下，该方法不一定比折半查找要好

```java
/**
 * 插值查找
 *
 * @param arr 已排序的数组
 * @param left 开始位置，如：0
 * @param right 结束位置，如：array.length-1
 * @param findValue
 * @return
 */
public static int insertValueSearch(int[] arr, int left, int right, int findValue) {
        //注意：findVal < arr[0]  和  findVal > arr[arr.length - 1] 必须需要, 否则我们得到的 mid 可能越界
        if (left > right || findValue < arr[0] || findValue > arr[arr.length - 1]) {
            return -1;
        }

        // 求出mid, 自适应
        int mid = left + (right - left) * (findValue - arr[left]) / (arr[right] - arr[left]);
        int midValue = arr[mid];
        if (findValue > midValue) {
            // 向右递归
            return insertValueSearch(arr, mid + 1, right, findValue);
        } else if (findValue < midValue) {
            // 向左递归
            return insertValueSearch(arr, left, mid - 1, findValue);
        } else {
            return mid;
        }
}
```



### 斐波那契查找

黄金分割点是指把一条线段分割为两部分，使其中一部分与全长之比等于另一部分与这部分之比。取其前三位数字的近似值是0.618。由于按此比例设计的造型十分美丽，因此称为黄金分割，也称为中外比。这是一个神奇的数字，会带来意向不大的效果。斐波那契数列{1, 1,2, 3, 5, 8, 13,21, 34, 55 }发现斐波那契数列的两个相邻数的比例，无限接近黄金分割值0.618。
斐波那契查找原理与前两种相似，仅仅改变了中间结点(mid)的位置，mid不再是中间或插值得到，而是位于黄金分割点附近，即mid=low+F(k-1)-1(F代表斐波那契数列)，如下图所示：

![斐波那契查找](png/Java/算法-斐波那契查找.png)

JAVA代码如下：

```java
/**
 * 因为后面我们mid=low+F(k-1)-1，需要使用到斐波那契数列，因此我们需要先获取到一个斐波那契数列
 * <p>
 * 非递归方法得到一个斐波那契数列
 *
 * @return
 */
private static int[] getFibonacci() {
        int[] fibonacci = new int[20];
        fibonacci[0] = 1;
        fibonacci[1] = 1;
        for (int i = 2; i < fibonacci.length; i++) {
            fibonacci[i] = fibonacci[i - 1] + fibonacci[i - 2];
        }
        return fibonacci;
}

/**
 * 编写斐波那契查找算法
 * <p>
 * 使用非递归的方式编写算法
 *
 * @param arr       数组
 * @param findValue 我们需要查找的关键码(值)
 * @return 返回对应的下标，如果没有-1
 */
public static int fibonacciSearch(int[] arr, int findValue) {
        int low = 0;
        int high = arr.length - 1;
        int k = 0;// 表示斐波那契分割数值的下标
        int mid = 0;// 存放mid值
        int[] fibonacci = getFibonacci();// 获取到斐波那契数列
        // 获取到斐波那契分割数值的下标
        while (high > fibonacci[k] - 1) {
            k++;
        }

        // 因为 fibonacci[k] 值可能大于 arr 的 长度，因此我们需要使用Arrays类，构造一个新的数组
        int[] temp = Arrays.copyOf(arr, fibonacci[k]);
        // 实际上需求使用arr数组最后的数填充 temp
        for (int i = high + 1; i < temp.length; i++) {
            temp[i] = arr[high];
        }

        // 使用while来循环处理，找到我们的数 findValue
        while (low <= high) {
            mid = low + fibonacci[k] - 1;
            if (findValue < temp[mid]) {
                high = mid - 1;
                k--;
            } else if (findValue > temp[mid]) {
                low = mid + 1;
                k++;
            } else {
                return Math.min(mid, high);
            }
        }

        return -1;
}
```



## 搜索算法

### 深度优先搜索(DFS)

深度优先搜索（Depth-First Search / DFS）是一种**优先遍历子节点**而不是回溯的算法。

![深度优先搜索](png/Java/算法-深度优先搜索.jpg)

**DFS解决的是连通性的问题**。即给定两个点，一个是起始点，一个是终点，判断是不是有一条路径能从起点连接到终点。起点和终点，也可以指的是某种起始状态和最终的状态。问题的要求并不在乎路径是长还是短，只在乎有还是没有。



**代码案例**

```java
/**
 * Depth-First Search(DFS)
 * <p>
 * 从根节点出发，沿着左子树方向进行纵向遍历，直到找到叶子节点为止。然后回溯到前一个节点，进行右子树节点的遍历，直到遍历完所有可达节点为止。
 * <p>
 * 数据结构：栈
 * 父节点入栈，父节点出栈，先右子节点入栈，后左子节点入栈。递归遍历全部节点即可
 *
 * @author lry
 */
public class DepthFirstSearch {

    /**
     * 树节点
     *
     * @param <V>
     */
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class TreeNode<V> {
        private V value;
        private List<TreeNode<V>> childList;

        // 二叉树节点支持如下

        public TreeNode<V> getLeft() {
            if (childList == null || childList.isEmpty()) {
                return null;
            }

            return childList.get(0);

        }

        public TreeNode<V> getRight() {
            if (childList == null || childList.isEmpty()) {
                return null;
            }

            return childList.get(1);
        }
    }

    /**
     * 模型：
     * .......A
     * ...../   \
     * ....B     C
     * .../ \   /  \
     * ..D   E F    G
     * ./ \        /  \
     * H  I       J    K
     */
    public static void main(String[] args) {
        TreeNode<String> treeNodeA = new TreeNode<>("A", new ArrayList<>());
        TreeNode<String> treeNodeB = new TreeNode<>("B", new ArrayList<>());
        TreeNode<String> treeNodeC = new TreeNode<>("C", new ArrayList<>());
        TreeNode<String> treeNodeD = new TreeNode<>("D", new ArrayList<>());
        TreeNode<String> treeNodeE = new TreeNode<>("E", new ArrayList<>());
        TreeNode<String> treeNodeF = new TreeNode<>("F", new ArrayList<>());
        TreeNode<String> treeNodeG = new TreeNode<>("G", new ArrayList<>());
        TreeNode<String> treeNodeH = new TreeNode<>("H", new ArrayList<>());
        TreeNode<String> treeNodeI = new TreeNode<>("I", new ArrayList<>());
        TreeNode<String> treeNodeJ = new TreeNode<>("J", new ArrayList<>());
        TreeNode<String> treeNodeK = new TreeNode<>("K", new ArrayList<>());
        // A->B,C
        treeNodeA.getChildList().add(treeNodeB);
        treeNodeA.getChildList().add(treeNodeC);
        // B->D,E
        treeNodeB.getChildList().add(treeNodeD);
        treeNodeB.getChildList().add(treeNodeE);
        // C->F,G
        treeNodeC.getChildList().add(treeNodeF);
        treeNodeC.getChildList().add(treeNodeG);
        // D->H,I
        treeNodeD.getChildList().add(treeNodeH);
        treeNodeD.getChildList().add(treeNodeI);
        // G->J,K
        treeNodeG.getChildList().add(treeNodeJ);
        treeNodeG.getChildList().add(treeNodeK);

        System.out.println("非递归方式");
        dfsNotRecursive(treeNodeA);
        System.out.println();
        System.out.println("前续遍历");
        dfsPreOrderTraversal(treeNodeA, 0);
        System.out.println();
        System.out.println("后续遍历");
        dfsPostOrderTraversal(treeNodeA, 0);
        System.out.println();
        System.out.println("中续遍历");
        dfsInOrderTraversal(treeNodeA, 0);
    }

    /**
     * 非递归方式
     *
     * @param tree
     * @param <V>
     */
    public static <V> void dfsNotRecursive(TreeNode<V> tree) {
        if (tree != null) {
            // 次数之所以用 Map 只是为了保存节点的深度，如果没有这个需求可以改为 Stack<TreeNode<V>>
            Stack<Map<TreeNode<V>, Integer>> stack = new Stack<>();
            Map<TreeNode<V>, Integer> root = new HashMap<>();
            root.put(tree, 0);
            stack.push(root);

            while (!stack.isEmpty()) {
                Map<TreeNode<V>, Integer> item = stack.pop();
                TreeNode<V> node = item.keySet().iterator().next();
                int depth = item.get(node);

                // 打印节点值以及深度
                System.out.print("-->[" + node.getValue().toString() + "," + depth + "]");

                if (node.getChildList() != null && !node.getChildList().isEmpty()) {
                    for (TreeNode<V> treeNode : node.getChildList()) {
                        Map<TreeNode<V>, Integer> map = new HashMap<>();
                        map.put(treeNode, depth + 1);
                        stack.push(map);
                    }
                }
            }
        }
    }

    /**
     * 递归前序遍历方式
     * <p>
     * 前序遍历(Pre-Order Traversal) ：指先访问根，然后访问子树的遍历方式，二叉树则为：根->左->右
     *
     * @param tree
     * @param depth
     * @param <V>
     */
    public static <V> void dfsPreOrderTraversal(TreeNode<V> tree, int depth) {
        if (tree != null) {
            // 打印节点值以及深度
            System.out.print("-->[" + tree.getValue().toString() + "," + depth + "]");

            if (tree.getChildList() != null && !tree.getChildList().isEmpty()) {
                for (TreeNode<V> item : tree.getChildList()) {
                    dfsPreOrderTraversal(item, depth + 1);
                }
            }
        }
    }

    /**
     * 递归后序遍历方式
     * <p>
     * 后序遍历(Post-Order Traversal)：指先访问子树，然后访问根的遍历方式，二叉树则为：左->右->根
     *
     * @param tree
     * @param depth
     * @param <V>
     */
    public static <V> void dfsPostOrderTraversal(TreeNode<V> tree, int depth) {
        if (tree != null) {
            if (tree.getChildList() != null && !tree.getChildList().isEmpty()) {
                for (TreeNode<V> item : tree.getChildList()) {
                    dfsPostOrderTraversal(item, depth + 1);
                }
            }

            // 打印节点值以及深度
            System.out.print("-->[" + tree.getValue().toString() + "," + depth + "]");
        }
    }

    /**
     * 递归中序遍历方式
     * <p>
     * 中序遍历(In-Order Traversal)：指先访问左（右）子树，然后访问根，最后访问右（左）子树的遍历方式，二叉树则为：左->根->右
     *
     * @param tree
     * @param depth
     * @param <V>
     */
    public static <V> void dfsInOrderTraversal(TreeNode<V> tree, int depth) {
        if (tree.getLeft() != null) {
            dfsInOrderTraversal(tree.getLeft(), depth + 1);
        }

        // 打印节点值以及深度
        System.out.print("-->[" + tree.getValue().toString() + "," + depth + "]");

        if (tree.getRight() != null) {
            dfsInOrderTraversal(tree.getRight(), depth + 1);
        }
    }

}
```



### 广度优先搜索(BFS)

广度优先搜索（Breadth-First Search / BFS）是**优先遍历邻居节点**而不是子节点的图遍历算法。

![广度优先搜索](png/Java/算法-广度优先搜索.jpg)

**BFS一般用来解决最短路径的问题**。和深度优先搜索不同，广度优先的搜索是从起始点出发，一层一层地进行，每层当中的点距离起始点的步数都是相同的，当找到了目的地之后就可以立即结束。广度优先的搜索可以同时从起始点和终点开始进行，称之为双端 BFS。这种算法往往可以大大地提高搜索的效率。



**代码案例**

```java
/**
 * Breadth-First Search(BFS)
 * <p>
 * 从根节点出发，在横向遍历二叉树层段节点的基础上纵向遍历二叉树的层次。
 * <p>
 * 数据结构：队列
 * 父节点入队，父节点出队列，先左子节点入队，后右子节点入队。递归遍历全部节点即可
 *
 * @author lry
 */
public class BreadthFirstSearch {

    /**
     * 树节点
     *
     * @param <V>
     */
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class TreeNode<V> {
        private V value;
        private List<TreeNode<V>> childList;
    }

    /**
     * 模型：
     * .......A
     * ...../   \
     * ....B     C
     * .../ \   /  \
     * ..D   E F    G
     * ./ \        /  \
     * H  I       J    K
     */
    public static void main(String[] args) {
        TreeNode<String> treeNodeA = new TreeNode<>("A", new ArrayList<>());
        TreeNode<String> treeNodeB = new TreeNode<>("B", new ArrayList<>());
        TreeNode<String> treeNodeC = new TreeNode<>("C", new ArrayList<>());
        TreeNode<String> treeNodeD = new TreeNode<>("D", new ArrayList<>());
        TreeNode<String> treeNodeE = new TreeNode<>("E", new ArrayList<>());
        TreeNode<String> treeNodeF = new TreeNode<>("F", new ArrayList<>());
        TreeNode<String> treeNodeG = new TreeNode<>("G", new ArrayList<>());
        TreeNode<String> treeNodeH = new TreeNode<>("H", new ArrayList<>());
        TreeNode<String> treeNodeI = new TreeNode<>("I", new ArrayList<>());
        TreeNode<String> treeNodeJ = new TreeNode<>("J", new ArrayList<>());
        TreeNode<String> treeNodeK = new TreeNode<>("K", new ArrayList<>());
        // A->B,C
        treeNodeA.getChildList().add(treeNodeB);
        treeNodeA.getChildList().add(treeNodeC);
        // B->D,E
        treeNodeB.getChildList().add(treeNodeD);
        treeNodeB.getChildList().add(treeNodeE);
        // C->F,G
        treeNodeC.getChildList().add(treeNodeF);
        treeNodeC.getChildList().add(treeNodeG);
        // D->H,I
        treeNodeD.getChildList().add(treeNodeH);
        treeNodeD.getChildList().add(treeNodeI);
        // G->J,K
        treeNodeG.getChildList().add(treeNodeJ);
        treeNodeG.getChildList().add(treeNodeK);

        System.out.println("递归方式");
        bfsRecursive(Arrays.asList(treeNodeA), 0);
        System.out.println();
        System.out.println("非递归方式");
        bfsNotRecursive(treeNodeA);
    }

    /**
     * 递归遍历
     *
     * @param children
     * @param depth
     * @param <V>
     */
    public static <V> void bfsRecursive(List<TreeNode<V>> children, int depth) {
        List<TreeNode<V>> thisChildren, allChildren = new ArrayList<>();
        for (TreeNode<V> child : children) {
            // 打印节点值以及深度
            System.out.print("-->[" + child.getValue().toString() + "," + depth + "]");

            thisChildren = child.getChildList();
            if (thisChildren != null && thisChildren.size() > 0) {
                allChildren.addAll(thisChildren);
            }
        }

        if (allChildren.size() > 0) {
            bfsRecursive(allChildren, depth + 1);
        }
    }

    /**
     * 非递归遍历
     *
     * @param tree
     * @param <V>
     */
    public static <V> void bfsNotRecursive(TreeNode<V> tree) {
        if (tree != null) {
            // 跟上面一样，使用 Map 也只是为了保存树的深度，没这个需要可以不用 Map
            Queue<Map<TreeNode<V>, Integer>> queue = new ArrayDeque<>();
            Map<TreeNode<V>, Integer> root = new HashMap<>();
            root.put(tree, 0);
            queue.offer(root);

            while (!queue.isEmpty()) {
                Map<TreeNode<V>, Integer> itemMap = queue.poll();
                TreeNode<V> node = itemMap.keySet().iterator().next();
                int depth = itemMap.get(node);

                //打印节点值以及深度
                System.out.print("-->[" + node.getValue().toString() + "," + depth + "]");

                if (node.getChildList() != null && !node.getChildList().isEmpty()) {
                    for (TreeNode<V> child : node.getChildList()) {
                        Map<TreeNode<V>, Integer> map = new HashMap<>();
                        map.put(child, depth + 1);
                        queue.offer(map);
                    }
                }
            }
        }
    }

}
```



### 迪杰斯特拉算法(Dijkstra)

**迪杰斯特拉(Dijkstra)算法** 是典型最短路径算法，用于计算一个节点到其他节点的最短路径。它的主要特点是以起始点为中心向外层层扩展(广度优先搜索思想)，直到扩展到终点为止。



**基本思想**

通过Dijkstra计算图G中的最短路径时，需要指定起点s(即从顶点s开始计算)。

此外，引进两个集合S和U。S的作用是记录已求出最短路径的顶点(以及相应的最短路径长度)，而U则是记录还未求出最短路径的顶点(以及该顶点到起点s的距离)。

初始时，S中只有起点s；U中是除s之外的顶点，并且U中顶点的路径是"起点s到该顶点的路径"。然后，从U中找出路径最短的顶点，并将其加入到S中；接着，更新U中的顶点和顶点对应的路径。 然后，再从U中找出路径最短的顶点，并将其加入到S中；接着，更新U中的顶点和顶点对应的路径。 ... 重复该操作，直到遍历完所有顶点。


**操作步骤**

- 初始时，S只包含起点s；U包含除s外的其他顶点，且U中顶点的距离为"起点s到该顶点的距离"[例如，U中顶点v的距离为(s,v)的长度，然后s和v不相邻，则v的距离为∞]
- U中选出"距离最短的顶点k"，并将顶点k加入到S中；同时，从U中移除顶点k
- 更新U中各个顶点到起点s的距离。之所以更新U中顶点的距离，是由于上一步中确定了k是求出最短路径的顶点，从而可以利用k来更新其它顶点的距离；例如，(s,v)的距离可能大于(s,k)+(k,v)的距离
- 重复步骤(2)和(3)，直到遍历完所有顶点



**迪杰斯特拉算法图解**

![img](png/Java/算法-迪杰斯特拉.png)

以上图G4为例，来对迪杰斯特拉进行算法演示(以第4个顶点D为起点)：

![img](png/Java/算法-迪杰斯特拉演示.png)

 **代码案例**

```java
public class Dijkstra {
     // 代表正无穷
    public static final int M = 10000;
    public static String[] names = new String[]{"A", "B", "C", "D", "E", "F", "G",};

    public static void main(String[] args) {
        // 二维数组每一行分别是 A、B、C、D、E 各点到其余点的距离,
        // A -> A 距离为0, 常量M 为正无穷
        int[][] weight1 = {
                {0, 12, M, M, M, 16, 14},
                {12, 0, 10, M, M, 7, M},
                {M, 10, 0, 3, 5, 6, M},
                {M, M, 3, 0, 4, M, M},
                {M, M, 5, 4, 0, 2, 8},
                {16, 7, 6, M, 2, 0, 9},
                {14, M, M, M, 8, 9, 0}
        };

        int start = 0;
        int[] shortPath = dijkstra(weight1, start);
        System.out.println("===============");
        for (int i = 0; i < shortPath.length; i++) {
            System.out.println("从" + names[start] + "出发到" + names[i] + "的最短距离为：" + shortPath[i]);
        }
    }

    /**
     * Dijkstra算法
     *
     * @param weight 图的权重矩阵
     * @param start  起点编号start（从0编号，顶点存在数组中）
     * @return 返回一个int[] 数组，表示从start到它的最短路径长度
     */
    public static int[] dijkstra(int[][] weight, int start) {
        // 顶点个数
        int n = weight.length;
        // 标记当前该顶点的最短路径是否已经求出,1表示已求出
        int[] visited = new int[n];
        // 保存start到其他各点的最短路径
        int[] shortPath = new int[n];

        // 保存start到其他各点最短路径的字符串表示
        String[] path = new String[n];
        for (int i = 0; i < n; i++) {
            path[i] = names[start] + "-->" + names[i];
        }

        // 初始化，第一个顶点已经求出
        shortPath[start] = 0;
        visited[start] = 1;

        // 要加入n-1个顶点
        for (int count = 1; count < n; count++) {
            // 选出一个距离初始顶点start最近的未标记顶点
            int k = -1;
            int dMin = Integer.MAX_VALUE;
            for (int i = 0; i < n; i++) {
                if (visited[i] == 0 && weight[start][i] < dMin) {
                    dMin = weight[start][i];
                    k = i;
                }
            }

            // 将新选出的顶点标记为已求出最短路径，且到start的最短路径就是dmin
            shortPath[k] = dMin;
            visited[k] = 1;

            // 以k为中间点，修正从start到未访问各点的距离
            for (int i = 0; i < n; i++) {
                // 如果 '起始点到当前点距离' + '当前点到某点距离' < '起始点到某点距离', 则更新
                if (visited[i] == 0 && weight[start][k] + weight[k][i] < weight[start][i]) {
                    weight[start][i] = weight[start][k] + weight[k][i];
                    path[i] = path[k] + "-->" + names[i];
                }
            }
        }

        for (int i = 0; i < n; i++) {
            System.out.println("从" + names[start] + "出发到" + names[i] + "的最短路径为：" + path[i]);
        }

        return shortPath;
    }

}
```



### kruskal(克鲁斯卡尔)算法

https://www.cnblogs.com/skywang12345/category/508186.html



## 排序算法

十种常见排序算法可以分为两大类：

- **非线性时间比较类排序**

  通过比较来决定元素间的相对次序，由于其时间复杂度不能突破O(nlogn)，因此称为非线性时间比较类排序。

- **线性时间非比较类排序**

  不通过比较来决定元素间的相对次序，它可以突破基于比较排序的时间下界，以线性时间运行，因此称为线性时间非比较类排序。

交换排序法 
   冒泡排序 |鸡尾酒排序 |奇偶排序 |梳排序 |地精排序(gnome_sort) |Bogo排序|快速排序
选择排序法 
   选择排序 | 堆排序
插入排序法 
   插入排序 | 希尔排序 | 二叉查找树排序 | Library sort | Patience sorting
归并排序法 
   归并排序 | Strand sort
非比较排序法 
   基数排序 | 桶排序 | 计数排序 | 鸽巢排序 | Burstsort | Bead sort
其他 
   拓扑排序 | 排序网络 | Bitonic sorter | Batcher odd-even mergesort | Pancake sorting
低效排序法 
   Bogosort | Stooge sort

![排序算法](png/Java/排序算法.png)

### 冒泡排序（Bubble Sort）

**循环遍历多次每次从前往后把大元素往后调，每次确定一个最大(最小)元素，多次后达到排序序列。**这个算法的名字由来是因为越小的元素会经由交换慢慢“浮”到数列的顶端。 

![冒泡排序](png/Java/冒泡排序.jpg)

![冒泡排序](png/Java/冒泡排序.gif)

**算法步骤**

- 比较相邻的元素。如果第一个比第二个大，就交换它们两个
- 对每一对相邻元素作同样的工作，从开始第一对到结尾的最后一对，这样在最后的元素应该会是最大的数
- 针对所有的元素重复以上的步骤，除了最后一个
- 重复步骤1~3，直到排序完成

**代码实现**

```java
/**
  * 冒泡排序
  * <p>
  * 描述：每轮连续比较相邻的两个数，前数大于后数，则进行替换。每轮完成后，本轮最大值已被移至最后
  *
  * @param arr 待排序数组
  */
public static int[] bubbleSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        for (int j = 0; j < arr.length - 1 - i; j++) {
            // 每次比较2个相邻的数,前一个小于后一个
            if (arr[j] > arr[j + 1]) {
                int tmp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = tmp;
            }
        }
    }
    
    return arr;
}
```

以下是冒泡排序算法复杂度:

| 平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度 |
| :------------- | :------- | :------- | :--------- |
| O(n²)          | O(n)     | O(n²)    | O(1)       |

冒泡排序是最容易实现的排序, 最坏的情况是每次都需要交换, 共需遍历并交换将近n²/2次, 时间复杂度为O(n²). 最佳的情况是内循环遍历一次后发现排序是对的, 因此退出循环, 时间复杂度为O(n)。平均来讲, 时间复杂度为O(n²). 由于冒泡排序中只有缓存的temp变量需要内存空间, 因此空间复杂度为常量O(1)。

Tips: 由于冒泡排序只在相邻元素大小不符合要求时才调换他们的位置, 它并不改变相同元素之间的相对顺序, 因此它是稳定的排序算法。

#### 鸡尾酒排序(Cocktail sort)

**鸡尾酒排序**，也就是**定向冒泡排序**, 鸡尾酒搅拌排序, 是冒泡排序的一种变形。此算法与冒泡排序的不同处在于排序时是以双向在序列中进行排序。此算法与冒泡排序的不同处在于**从低到高然后从高到低**，而冒泡排序则仅从低到高去比较序列里的每个元素。他可以得到比冒泡排序稍微好一点的效能。

**算法步骤**

 i. 先对数组从左到右进行升序的冒泡排序；
 ii. 再对数组进行从右到左的降序的冒泡排序；
 iii. 以此类推，持续的、依次的改变冒泡的方向，并不断缩小没有排序的数组范围；

**算法实例**

例：     88   7   79   64   55   98   48   52   4   13
第一趟:   7   79  64   55   88   48   52   4    13  98
第二趟：  4   7   79   64   55   88   48   42   13   98
第三趟：  4   7   64   55   79   48   42   13   88  98
第四趟：  4   7   13   64   55   79   48   42   88  98
第五趟：  4   7   13   55   64   48  42   79   88  98
第六趟：  4   7   13   42   55   64  48   79   88  98
第七趟：  4   7   13   42   55   48  64   79   88  98
第八趟：  4   7   13   42   48   55  64   79   88  98  

鸡尾酒排序的代码如下：

```
public static void coktailSort(int[] arr) {

        for (int i = 0; i < arr.length/2; i++) {
            for (int j = i; j < arr.length-1-i; j++) {
                if(arr[j]>arr[j+1]) {
                    int t=arr[j];
                    arr[j]=arr[j+1];
                    arr[j+1]=t;
                }
            }
            for (int j = arr.length-(1+1)-i; j >i; j--) {
                if(arr[j]<arr[j-1]) {
                    int t=arr[j];
                    arr[j]=arr[j-1];
                    arr[j-1]=t;
                }
            }
        }

    }
```

与冒泡排序不同的地方：

鸡尾酒排序等于是冒泡排序的轻微变形。不同的地方在于从低到高然后从高到低，而冒泡排序则仅从低到高去比较序列里的每个元素。他可以得到比冒泡排序稍微好一点的效能，原因是冒泡排序只从一个方向进行比对(由低到高)，每次循环只移动一个项目。

以序列(2,3,4,5,1)为例，鸡尾酒排序只需要访问一次序列就可以完成排序，但如果使用冒泡排序则需要四次。 但是在乱数序列的状态下，鸡尾酒排序与冒泡排序的效率都很差劲。

#### 地精排序(gnomeSort)

地精排序（侏儒排序）,传说有一个地精在排列一排花盘。他从前至后的排列，如果相邻的两个花盘顺序正确，他向前一步；如果花盘顺序错误，他后退一步，直到所有的花盘的顺序都排列好。(地精排序思路与插入排序和冒泡排序很像, 主要其对代码进行了极简化)

稳定性:稳定

简单，只有一层循环。时间复杂度O(n^2)，最优复杂度O(n),平均时间复杂度O(n^2)

```
public static void gnomeSort(int[] arr) {

        int i=1;
        while(i<arr.length) {

            if(i==0||arr[i]>=arr[i-1]) {
                i++;
            }else {
                int t=arr[i];
                arr[i]=arr[i-1];
                arr[i-1]=t;
                i--;
            }

        }

    }
```

#### 奇偶排序(OddevenSort)

奇偶排序或奇偶换位排序，或砖排序，是一种相对简单的排序算法，最初发明用于有本地互连的并行计算。这是与冒泡排序特点类似的一种比较排序。

该算法中，排序过程分两个阶段，奇交换和偶交换，两种交换都是成对出现。对于奇交换，它总是比较奇数索引以及其相邻的后续元素。对偶交换，总是比较偶数索引和其相邻的后续元素。思路是在数组中重复两趟扫描。第一趟扫描选择所有的数据项对，a[j]和a[j+1]，j是奇数(j=1, 3, 5……)。如果它们的关键字的值次序颠倒，就交换它们。第二趟扫描对所有的偶数数据项进行同样的操作(j=2, 4,6……)。重复进行这样两趟的排序直到数组全部有序。

简单来说就是：奇数列排一趟序,偶数列排一趟序,再奇数排,再偶数排,直到全部有序。

**示例：**

待排数组[6 2 4 1 5 9]  //目标按从小到大排序

第一次比较奇数列,奇数列与它的邻居偶数列比较,如6和2比,4和1比,5和9比

[6 2 4 1 5 9]

交换后变成

[2 6 1 4 5 9]

第二次比较偶数列,即6和1比,5和5比

[2 6 1 4 5 9]

交换后变成

[2 1 6 4 5 9]

第三次又是奇数列,选择的是2,6,5分别与它们的邻居列比较

[2 1 6 4 5 9]

交换后

[1 2 4 6 5 9] 

第四次偶数列

[1 2 4 6 5 9]

一次交换

[1 2 4 5 6 9]

```
/*
 * 奇偶排序
 */
public class OddevenSort {
    public static void main(String[] args) {
        int[] arrayData = { 2, 3, 4, 5, 6, 7, 8, 9, 1 };
        OddevenSortMethod(arrayData);
        for (int integer : arrayData) {
            System.out.print(integer);
            System.out.print(" ");
        }
    }
 
    public static void OddevenSortMethod(int[] arrayData) {
        int temp;
        int length = arrayData.length;
        boolean whetherSorted = true;
 
        while (whetherSorted) {
            whetherSorted = false;
            for (int i = 1; i < length - 1; i++) {
                if (arrayData[i] > arrayData[i + 1]) {
                    temp = arrayData[i];
                    arrayData[i] = arrayData[i + 1];
                    arrayData[i + 1] = temp;
                    whetherSorted = true;
                }
            }
 
            for (int i = 0; i < length - 1; i++) {
                if (arrayData[i] > arrayData[i + 1]) {
                    temp = arrayData[i];
                    arrayData[i] = arrayData[i + 1];
                    arrayData[i + 1] = temp;
                    whetherSorted = true;
                }
            }
        }
    }
}
```

算法复杂度分析

| **排序方法** | **时间复杂度** | **空间复杂度** | **稳定性** | **复杂性** |      |        |
| ------------ | -------------- | -------------- | ---------- | ---------- | ---- | ------ |
| **平均情况** | **最坏情况**   | **最好情况**   |            |            |      |        |
| **奇偶排序** | O(nlog2n)      | O(nlog2n)      | O(n)       | O(1)       | 稳定 | 较简单 |

### 选择排序（Selection Sort）

**选择排序(SelectionSort)**是一种简单直观的排序算法。它的工作原理：首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。 

![选择排序](png/Java/选择排序.jpg)

![选择排序](png/Java/选择排序.gif)

**算法描述**

n个记录的直接选择排序可经过n-1趟直接选择排序得到有序结果。具体算法描述如下：

- 初始状态：无序区为R[1..n]，有序区为空
- 第i趟排序(i=1,2,3…n-1)开始时，当前有序区和无序区分别为R[1..i-1]和R(i..n）。该趟排序从当前无序区中-选出关键字最小的记录 R[k]，将它与无序区的第1个记录R交换，使R[1..i]和R[i+1..n)分别变为记录个数增加1个的新有序区和记录个数减少1个的新无序区
- n-1趟结束，数组有序化了

**代码实现**

```java
 /**
     * 选择排序
     * <p>
     * 描述：每轮选择出最小值，然后依次放置最前面
     *
     * @param arr 待排序数组
     */
    public static int[] selectSort(int[] arr) {
        for (int i = 0; i < arr.length - 1; i++) {
            // 选最小的记录
            int min = i;
            for (int j = i + 1; j < arr.length; j++) {
                if (arr[min] > arr[j]) {
                    min = j;
                }
            }

            // 内层循环结束后，即找到本轮循环的最小的数以后，再进行交换：交换a[i]和a[min]
            if (min != i) {
                int temp = arr[i];
                arr[i] = arr[min];
                arr[min] = temp;
            }
        }

        return arr;
    }
```

以下是选择排序复杂度:

| 平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度 |
| :------------- | :------- | :------- | :--------- |
| O(n²)          | O(n²)    | O(n²)    | O(1)       |

选择排序的时间复杂度：简单选择排序的比较次数与序列的初始排序无关。 假设待排序的序列有 N 个元素，则比较次数永远都是N (N - 1) / 2。而移动次数与序列的初始排序有关。当序列正序时，移动次数最少，为 0。当序列反序时，移动次数最多，为3N (N - 1) / 2。

选择排序的简单和直观名副其实，这也造就了它”出了名的慢性子”，无论是哪种情况，哪怕原数组已排序完成，它也将花费将近n²/2次遍历来确认一遍。即便是这样，它的排序结果也还是不稳定的。 唯一值得高兴的是，它并不耗费额外的内存空间。

### 插入排序（Insertion Sort）

插入排序（InsertionSort）的算法描述是一种简单直观的排序算法。它的工作原理是通过构建有序序列，对于未排序数据，在已排序序列中从后向前扫描，找到相应位置并插入。插入排序由于操作不尽相同，可分为 **直接插入排序、折半插入排序(又称二分插入排序)、链表插入排序、希尔排序** 。

![插入排序](png/Java/插入排序.jpg)

![插入排序](png/Java/插入排序.gif)

**算法描述**

一般来说，插入排序都采用in-place在数组上实现。具体算法描述如下：

- 从第一个元素开始，该元素可以认为已经被排序
- 取出下一个元素，在已经排序的元素序列中从后向前扫描
- 如果该元素（已排序）大于新元素，将该元素移到下一位置
- 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
- 将新元素插入到该位置后
- 重复步骤2~5

**代码实现**

```java
   /**
     * 直接插入排序
     * <p>
     * 1. 从第一个元素开始，该元素可以认为已经被排序
     * 2. 取出下一个元素，在已经排序的元素序列中从后向前扫描
     * 3. 如果该元素（已排序）大于新元素，将该元素移到下一位置
     * 4. 重复步骤3，直到找到已排序的元素小于或者等于新元素的位置
     * 5. 将新元素插入到该位置后
     * 6. 重复步骤2~5
     *
     * @param arr 待排序数组
     */
    public static int[] insertionSort(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            // 取出下一个元素，在已经排序的元素序列中从后向前扫描
            int temp = arr[i];
            for (int j = i; j >= 0; j--) {
                if (j > 0 && arr[j - 1] > temp) {
                    // 如果该元素（已排序）大于取出的元素temp，将该元素移到下一位置
                    arr[j] = arr[j - 1];
                } else {
                    // 将新元素插入到该位置后
                    arr[j] = temp;
                    break;
                }
            }
        }

        return arr;
    }

    /**
     * 折半插入排序
     * <p>
     * 往前找合适的插入位置时采用二分查找的方式，即折半插入
     * <p>
     * 交换次数较多的实现
     *
     * @param arr 待排序数组
     */
    public static int[] insertionBinarySort(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] < arr[i - 1]) {
                int tmp = arr[i];

                // 记录搜索范围的左边界,右边界
                int low = 0, high = i - 1;
                while (low <= high) {
                    // 记录中间位置Index
                    int mid = (low + high) / 2;
                    // 比较中间位置数据和i处数据大小，以缩小搜索范围
                    if (arr[mid] < tmp) {
                        // 左边指针则一只中间位置+1
                        low = mid + 1;
                    } else {
                        // 右边指针则一只中间位置-1
                        high = mid - 1;
                    }
                }

                // 将low~i处数据整体向后移动1位
                for (int j = i; j > low; j--) {
                    arr[j] = arr[j - 1];
                }
                arr[low] = tmp;
            }
        }

        return arr;
    }
```

插入排序复杂度：

| 平均时间复杂度 | 最好情况 | 最坏情况 | 空间复杂度 |
| :------------- | :------- | :------- | :--------- |
| O(n²)          | O(n)     | O(n²)    | O(1)       |

Tips：由于直接插入排序每次只移动一个元素的位， 并不会改变值相同的元素之间的排序， 因此它是一种稳定排序。插入排序在实现上，通常采用in-place排序（即只需用到O(1)的额外空间的排序），因而在从后向前扫描过程中，需要反复把已排序元素逐步向后挪位，为最新元素提供插入空间。



### 希尔排序（Shell Sort）

1959年Shell发明，第一个突破O(n2)的排序算法。希尔排序(Shell Sort)是插入排序的一种，是针对直接插入排序算法的改进，是将整个无序列分割成若干小的子序列分别进行插入排序，希尔排序并不稳定。它与插入排序的不同之处在于，它会优先比较距离较远的元素。希尔排序又叫**缩小增量排序**。



![希尔排序](png/Java/希尔排序.gif)

**算法原理**

先将整个待排序的记录序列分割成为若干子序列分别进行直接插入排序，具体算法描述：

- 选择一个增量序列t1，t2，…，tk，其中ti>tj，tk=1
- 按增量序列个数k，对序列进行k 趟排序
- 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序。仅增量因子为1 时，整个序列作为一个表来处理，表长度即为整个序列的长度

**算法实例**

假如有初始数据：25  11  45  26  12  78。

　　1、第一轮排序，将该数组分成 6/2=3 个数组序列，第1个数据和第4个数据为一对，第2个数据和第5个数据为一对，第3个数据和第6个数据为一对，每对数据进行比较排序，排序后顺序为：[25, 11, 45, 26, 12, 78]。

　　2、第二轮排序 ，将上轮排序后的数组分成6/4=1个数组序列，此时逐个对数据比较，按照插入排序对该数组进行排序，排序后的顺序为：[11, 12, 25, 26, 45, 78]。

　　对于插入排序而言，如果原数组是基本有序的，那排序效率就可大大提高。另外，对于数量较小的序列使用直接插入排序，会因需要移动的数据量少，其效率也会提高。因此，希尔排序具有较高的执行效率。

　　希尔排序并不稳定，O(1)的额外空间，时间复杂度为O(N*(logN)^2)。

**代码实现**

```
 /**
     * 希尔排序
     * <p>
     * 1. 选择一个增量序列t1，t2，…，tk，其中ti>tj，tk=1；（一般初次取数组半长，之后每次再减半，直到增量为1）
     * 2. 按增量序列个数k，对序列进行k 趟排序；
     * 3. 每趟排序，根据对应的增量ti，将待排序列分割成若干长度为m 的子序列，分别对各子表进行直接插入排序。
     * 仅增量因子为1 时，整个序列作为一个表来处理，表长度即为整个序列的长度。
     *
     * @param arr 待排序数组
     */
    public static int[] shellSort(int[] arr) {
        int gap = arr.length / 2;

        // 不断缩小gap，直到1为止
        for (; gap > 0; gap /= 2) {
            // 使用当前gap进行组内插入排序
            for (int j = 0; (j + gap) < arr.length; j++) {
                for (int k = 0; (k + gap) < arr.length; k += gap) {
                    if (arr[k] > arr[k + gap]) {
                        int temp = arr[k + gap];
                        arr[k + gap] = arr[k];
                        arr[k] = temp;
                    }
                }
            }
        }

        return arr;
    }


// 其他实现shellSort

public void shellSort(int[] arr) {
        // i表示希尔排序中的第n/2+1个元素（或者n/4+1）
        // j表示希尔排序中从0到n/2的元素（n/4）
        // r表示希尔排序中n/2+1或者n/4+1的值
        int i, j, r, tmp;
        // 划组排序
        for(r = arr.length / 2; r >= 1; r = r / 2) {
            for(i = r; i < arr.length; i++) {
                tmp = arr[i];
                j = i - r;
                // 一轮排序
                while(j >= 0 && tmp < arr[j]) {
                    arr[j+r] = arr[j];
                    j -= r;
                }
                arr[j+r] = tmp;
            }
         
        }
        return arr；
    }
```

以下是希尔排序复杂度:

| 平均时间复杂度 | 最好情况   | 最坏情况   | 空间复杂度 |
| :------------- | :--------- | :--------- | :--------- |
| O(nlog2 n)     | O(nlog2 n) | O(nlog2 n) | O(1)       |

Tips：希尔排序的核心在于间隔序列的设定。既可以提前设定好间隔序列，也可以动态的定义间隔序列。动态定义间隔序列的算法是《算法（第4版）》的合著者Robert Sedgewick提出的。　

### 归并排序（Merging Sort）

归并排序，算林届的新秀，引领着分治法的潮流。归并排序法（Merging Sort）是将两个（或两个以上）有序表合并成一个新的有序表，即把待排序序列分为若干个子序列，每个子序列是有序的，然后再把有序子序列**合并**为整体有序序列。

这种思想是一种算法设计的思想，很多问题都可以采用这种方式解决。映射到编程领域，其实就是递归的思想。因此在归并排序的算法中，将会出现递归调用。

**应用场景**：内存少的时候使用，可以进行**并行计算**的时候使用。

**算法步骤**

- 选择**相邻**两个数组成一个有序序列
- 选择相邻的两个有序序列组成一个有序序列
- 重复第二步，直到全部组成一个**有序**序列

![归并排序](png/Java/归并排序.jpg)

![归并排序](png/Java/归并排序.gif)

**算法描述**

**a.递归法**（假设序列共有n个元素）

①. 将序列每相邻两个数字进行归并操作，形成 floor(n/2)个序列，排序后每个序列包含两个元素；
②. 将上述序列再次归并，形成 floor(n/4)个序列，每个序列包含四个元素；
③. 重复步骤②，直到所有元素排序完毕。

**b.迭代法**

①. 申请空间，使其大小为两个已经排序序列之和，该空间用来存放合并后的序列
②. 设定两个指针，最初位置分别为两个已经排序序列的起始位置
③. 比较两个指针所指向的元素，选择相对小的元素放入到合并空间，并移动指针到下一位置
④. 重复步骤③直到某一指针到达序列尾
⑤. 将另一序列剩下的所有元素直接复制到合并序列尾

**代码实现**

```java
/**
     * 归并排序（递归）
     * <p>
     * ①. 将序列每相邻两个数字进行归并操作，形成 floor(n/2)个序列，排序后每个序列包含两个元素；
     * ②. 将上述序列再次归并，形成 floor(n/4)个序列，每个序列包含四个元素；
     * ③. 重复步骤②，直到所有元素排序完毕。
     *
     * @param arr 待排序数组
     */
    public static int[] mergeSort(int[] arr) {
        return mergeSort(arr, 0, arr.length - 1);
    }

    private static int[] mergeSort(int[] arr, int low, int high) {
        int center = (high + low) / 2;
        if (low < high) {
            // 递归，直到low==high，也就是数组已不能再分了，
            mergeSort(arr, low, center);
            mergeSort(arr, center + 1, high);

            // 当数组不能再分，开始归并排序
            mergeSort(arr, low, center, high);
        }

        return arr;
    }

    private static void mergeSort(int[] a, int low, int mid, int high) {
        int[] temp = new int[high - low + 1];
        int i = low, j = mid + 1, k = 0;

        // 把较小的数先移到新数组中
        while (i <= mid && j <= high) {
            if (a[i] < a[j]) {
                temp[k++] = a[i++];
            } else {
                temp[k++] = a[j++];
            }
        }

        // 把左边剩余的数移入数组
        while (i <= mid) {
            temp[k++] = a[i++];
        }

        // 把右边边剩余的数移入数组
        while (j <= high) {
            temp[k++] = a[j++];
        }

        // 把新数组中的数覆盖nums数组
        for (int x = 0; x < temp.length; x++) {
            a[x + low] = temp[x];
        }
    }
```

以下是归并排序算法复杂度:

| 平均时间复杂度 | 最好情况  | 最坏情况  | 空间复杂度 |
| :------------- | :-------- | :-------- | :--------- |
| O(nlog₂n)      | O(nlog₂n) | O(nlog₂n) | O(n)       |

从效率上看，归并排序可算是排序算法中的”佼佼者”. 假设数组长度为n，那么拆分数组共需logn，, 又每步都是一个普通的合并子数组的过程， 时间复杂度为O(n)， 故其综合时间复杂度为O(nlogn)。另一方面， 归并排序多次递归过程中拆分的子数组需要保存在内存空间， 其空间复杂度为O(n)。

Tips：和选择排序一样，归并排序的性能不受输入数据的影响，但表现比选择排序好的多，因为始终都是`O(nlogn）`的时间复杂度。代价是需要额外的内存空间。



### 快速排序（Quick Sort）

快速排序（Quicksort）是对冒泡排序的一种改进，借用了分治的思想，由C. A. R. Hoare在1962年提出。基本思想是通过一趟排序将待排记录分隔成独立的两部分，其中一部分记录的关键字均比另一部分的关键字小，则可分别对这两部分记录继续进行排序，以达到整个序列有序。

![快速排序](png/Java/快速排序.gif)

**算法描述**

快速排序使用分治法来把一个串（list）分为两个子串（sub-lists）。具体算法描述如下：

- 从数列中挑出一个元素，称为 “基准”（pivot）
- 重新排序数列，所有元素比基准值小的摆放在基准前面，所有元素比基准值大的摆在基准的后面（相同的数可以到任一边）。在这个分区退出之后，该基准就处于数列的中间位置。这个称为分区（partition）操作
- 递归地（recursive）把小于基准值元素的子数列和大于基准值元素的子数列排序

**算法实例**

场景：对 6 1 2 7 9 3 4 5 10 8 这 10 个数进行排序
思路：
先找一个基准数（一个用来参照的数），为了方便，我们选最左边的 6，希望将 >6 的放到 6 的右边，<6 的放到 6 左边。
如：3 1 2 5 4 6 9 7 10 8
先假设需要将 6 挪到的位置为 k，k 左边的数 <6，右边的数 >6

- 我们先从初始数列“6 1 2 7 9 3 4 5 10 8 ”的两端开始“探测 ”，先从右边往左找一个 <6 的数，再从左往右找一个 >6 的数，然后交换。我们用变量 i 和变量 j 指向序列的最左边和最右边。刚开始时最左边 i=0 指向 6，最右边 j=9 指向 8 ；
- 现在设置的基准数是最左边的数，所以序列先右往左移动（j–），当找到一个 <6 的数（5）就停下来。
  接着序列从左往右移动（i++），直到找到一个 >6 的数又停下来（7）；
- 两者交换，结果：6 1 2 5 9 3 4 7 10 8；
- j 的位置继续向左移动（友情提示：每次都必须先从 j 的位置出发），发现 4 满足要求，接着 i++ 发现 9 满足要求，交换后的结果：6 1 2 5 4 3 9 7 10 8；
- j 的位置继续向左移动（友情提示：每次都必须先从 j 的位置出发），发现 4 满足要求，接着 i++ 发现 9 满足要求，交换后的结果：6 1 2 5 4 3 9 7 10 8；
- 目前 j 指向的值为 9，i 指向的值为 4，j-- 发现 3 符合要求，接着 i++ 发现 i=j，说明这一轮移动结束啦。现在将基准数 6 和 3 进行交换，结果：3 1 2 5 4 6 9 7 10 8；现在 6 左边的数都是 <6 的，而右边的数都是 >6 的，但游戏还没结束
- 我们将 6 左边的数拿出来先：3 1 2 5 4，这次以 3 为基准数进行调整，使得 3 左边的数 ❤️，右边的数 >3，根据之前的模拟，这次的结果：2 1 3 5 4
- 再将 2 1 抠出来重新整理，得到的结果： 1 2
- 剩下右边的序列：9 7 10 8 也是这样来搞，最终的结果： 1 2 3 4 5 6 7 8 9 10

**代码实现**

```java
/**
     * 快速排序（递归）
     * <p>
     * ①. 从数列中挑出一个元素，称为"基准"（pivot）。
     * ②. 重新排序数列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面（相同的数可以到任一边）。在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。
     * ③. 递归地（recursively）把小于基准值元素的子数列和大于基准值元素的子数列排序。
     *
     * @param arr 待排序数组
     */
    public static int[] quickSort(int[] arr) {
        return quickSort(arr, 0, arr.length - 1);
    }

    private static int[] quickSort(int[] arr, int low, int high) {
        if (arr.length <= 0 || low >= high) {
            return arr;
        }

        int left = low;
        int right = high;

        // 挖坑1：保存基准的值
        int temp = arr[left];
        while (left < right) {
            // 坑2：从后向前找到比基准小的元素，插入到基准位置坑1中
            while (left < right && arr[right] >= temp) {
                right--;
            }
            arr[left] = arr[right];
            // 坑3：从前往后找到比基准大的元素，放到刚才挖的坑2中
            while (left < right && arr[left] <= temp) {
                left++;
            }
            arr[right] = arr[left];
        }
        // 基准值填补到坑3中，准备分治递归快排
        arr[left] = temp;
        quickSort(arr, low, left - 1);
        quickSort(arr, left + 1, high);

        return arr;
    }

    /**
     * 快速排序（非递归）
     * <p>
     * ①. 从数列中挑出一个元素，称为"基准"（pivot）。
     * ②. 重新排序数列，所有比基准值小的元素摆放在基准前面，所有比基准值大的元素摆在基准后面（相同的数可以到任一边）。在这个分区结束之后，该基准就处于数列的中间位置。这个称为分区（partition）操作。
     * ③. 把分区之后两个区间的边界（low和high）压入栈保存，并循环①、②步骤
     *
     * @param arr 待排序数组
     */
    public static int[] quickSortByStack(int[] arr) {
        Stack<Integer> stack = new Stack<>();

        // 初始状态的左右指针入栈
        stack.push(0);
        stack.push(arr.length - 1);
        while (!stack.isEmpty()) {
            // 出栈进行划分
            int high = stack.pop();
            int low = stack.pop();

            int pivotIdx = partition(arr, low, high);

            // 保存中间变量
            if (pivotIdx > low) {
                stack.push(low);
                stack.push(pivotIdx - 1);
            }
            if (pivotIdx < high && pivotIdx >= 0) {
                stack.push(pivotIdx + 1);
                stack.push(high);
            }
        }

        return arr;
    }

    private static int partition(int[] arr, int low, int high) {
        if (arr.length <= 0) return -1;
        if (low >= high) return -1;
        int l = low;
        int r = high;

        // 挖坑1：保存基准的值
        int pivot = arr[l];
        while (l < r) {
            // 坑2：从后向前找到比基准小的元素，插入到基准位置坑1中
            while (l < r && arr[r] >= pivot) {
                r--;
            }
            arr[l] = arr[r];
            // 坑3：从前往后找到比基准大的元素，放到刚才挖的坑2中
            while (l < r && arr[l] <= pivot) {
                l++;
            }
            arr[r] = arr[l];
        }

        // 基准值填补到坑3中，准备分治递归快排
        arr[l] = pivot;
        return l;
    }
```

以下是快速排序算法复杂度:

| 平均时间复杂度 | 最好情况  | 最坏情况 | 空间复杂度             |
| :------------- | :-------- | :------- | :--------------------- |
| O(nlog₂n)      | O(nlog₂n) | O(n²)    | O(1)（原地分区递归版） |

快速排序排序效率非常高。 虽然它运行最糟糕时将达到O(n²)的时间复杂度, 但通常平均来看, 它的时间复杂为O(nlogn), 比同样为O(nlogn)时间复杂度的归并排序还要快. 快速排序似乎更偏爱乱序的数列, 越是乱序的数列, 它相比其他排序而言, 相对效率更高。

Tips: 同选择排序相似, 快速排序每次交换的元素都有可能不是相邻的, 因此它有可能打破原来值为相同的元素之间的顺序. 因此, 快速排序并不稳定。



### 基数排序（Radix Sort）

基数排序是按照低位先排序，然后收集；再按照高位排序，然后再收集；依次类推，直到最高位。有时候有些属性是有优先级顺序的，先按低优先级排序，再按高优先级排序。最后的次序就是高优先级高的在前，高优先级相同的低优先级高的在前。

![基数排序](png/Java/基数排序.gif)

**算法描述**

- 取得数组中的最大数，并取得位数
- arr为原始数组，从最低位开始取每个位组成radix数组
- 对radix进行计数排序（利用计数排序适用于小范围数的特点）

**代码实现**

```java
/**
 * 基数排序（LSD 从低位开始）
 * <p>
 * 基数排序适用于：
 * (1)数据范围较小，建议在小于1000
 * (2)每个数值都要大于等于0
 * <p>
 * ①. 取得数组中的最大数，并取得位数；
 * ②. arr为原始数组，从最低位开始取每个位组成radix数组；
 * ③. 对radix进行计数排序（利用计数排序适用于小范围数的特点）；
 *
 * @param arr 待排序数组
 */
public static int[] radixSort(int[] arr) {
    // 取得数组中的最大数，并取得位数
    int max = 0;
    for (int item : arr) {
        if (max < item) {
            max = item;
        }
    }
    int maxDigit = 1;
    while (max / 10 > 0) {
        maxDigit++;
        max = max / 10;
    }

    // 申请一个桶空间
    int[][] buckets = new int[10][arr.length];
    int base = 10;

    // 从低位到高位，对每一位遍历，将所有元素分配到桶中
    for (int i = 0; i < maxDigit; i++) {
        // 存储各个桶中存储元素的数量
        int[] bktLen = new int[10];

        // 分配：将所有元素分配到桶中
        for (int value : arr) {
            int whichBucket = (value % base) / (base / 10);
            buckets[whichBucket][bktLen[whichBucket]] = value;
            bktLen[whichBucket]++;
        }

        // 收集：将不同桶里数据挨个捞出来,为下一轮高位排序做准备,由于靠近桶底的元素排名靠前,因此从桶底先捞
        int k = 0;
        for (int b = 0; b < buckets.length; b++) {
            for (int p = 0; p < bktLen[b]; p++) {
                arr[k++] = buckets[b][p];
            }
        }

        base *= 10;
    }

    return arr;
}
```

以下是基数排序算法复杂度，其中k为最大数的位数：

| 平均时间复杂度 | 最好情况   | 最坏情况   | 空间复杂度 |
| :------------- | :--------- | :--------- | :--------- |
| O(d*(n+r))     | O(d*(n+r)) | O(d*(n+r)) | O(n+r)     |

其中，**d 为位数，r 为基数，n 为原数组个数**。在基数排序中，因为没有比较操作，所以在复杂上，最好的情况与最坏的情况在时间上是一致的，均为 `O(d*(n + r))`。基数排序更适合用于对时间, 字符串等这些**整体权值未知的数据**进行排序。

Tips: 基数排序不改变相同元素之间的相对顺序，因此它是稳定的排序算法。



### 堆排序（Heap Sort）

对于堆排序，首先是建立在堆的基础上，堆是一棵完全二叉树，还要先认识下大根堆和小根堆，完全二叉树中所有节点均大于(或小于)它的孩子节点，所以这里就分为两种情况：

- 如果所有节点**「大于」**孩子节点值，那么这个堆叫做**「大根堆」**，堆的最大值在根节点
- 如果所有节点**「小于」**孩子节点值，那么这个堆叫做**「小根堆」**，堆的最小值在根节点

堆排序（Heapsort）是指利用堆这种数据结构所设计的一种排序算法。堆排序的过程就是将待排序的序列构造成一个堆，选出堆中最大的移走，再把剩余的元素调整成堆，找出最大的再移走，重复直至有序。

![大根堆-小根堆](png/Java/大根堆-小根堆.jpg)

**算法描述**

- 将初始待排序关键字序列(R1,R2….Rn)构建成大顶堆，此堆为初始的无序区
- 将堆顶元素R[1]与最后一个元素R[n]交换，此时得到新的无序区(R1,R2,……Rn-1)和新的有序区(Rn),且满足R[1,2…n-1]<=R[n]
- 由于交换后新的堆顶R[1]可能违反堆的性质，因此需要对当前无序区(R1,R2,……Rn-1)调整为新堆，然后再次将R[1]与无序区最后一个元素交换，得到新的无序区(R1,R2….Rn-2)和新的有序区(Rn-1,Rn)。不断重复此过程直到有序区的元素个数为n-1，则整个排序过程完成

**动图演示**

![堆排序](png/Java//堆排序.gif)

**代码实现**

```java
/**
     * 堆排序算法
     *
     * @param arr 待排序数组
     */
    public static int[] heapSort(int[] arr) {
        // 将无序序列构建成一个堆，根据升序降序需求选择大顶堆或小顶堆
        for (int i = arr.length / 2 - 1; i >= 0; i--) {
            adjustHeap(arr, i, arr.length);
        }

        // 将最大的节点放在堆尾，然后从根节点重新调整
        for (int j = arr.length - 1; j > 0; j--) {
            // 交换
            int temp = arr[j];
            arr[j] = arr[0];
            arr[0] = temp;

            // 完成将以i对应的非叶子结点的树调整成大顶堆
            adjustHeap(arr, 0, j);
        }

        return arr;
    }

    /**
     * 功能： 完成将以i对应的非叶子结点的树调整成大顶堆
     *
     * @param arr    待排序数组
     * @param i      表示非叶子结点在数组中索引
     * @param length 表示对多少个元素继续调整， length 是在逐渐的减少
     */
    private static void adjustHeap(int[] arr, int i, int length) {
        // 先取出当前元素的值，保存在临时变量
        int temp = arr[i];
        //开始调整。说明：1. k = i * 2 + 1 k 是 i结点的左子结点
        for (int k = i * 2 + 1; k < length; k = k * 2 + 1) {
            // 说明左子结点的值小于右子结点的值
            if (k + 1 < length && arr[k] < arr[k + 1]) {
                // k 指向右子结点
                k++;
            }

            // 如果子结点大于父结点
            if (arr[k] > temp) {
                // 把较大的值赋给当前结点
                arr[i] = arr[k];
                // i 指向 k,继续循环比较
                i = k;
            } else {
                break;
            }
        }

        //当for 循环结束后，我们已经将以i 为父结点的树的最大值，放在了 最顶(局部)。将temp值放到调整后的位置
        arr[i] = temp;
    }
```

| 平均时间复杂度 | 最好情况       | 最坏情况       | 空间复杂度 |
| :------------- | :------------- | :------------- | :--------- |
| O(n \log_{2}n) | O(n \log_{2}n) | O(n \log_{2}n) | O(1)       |

Tips: **由于堆排序中初始化堆的过程比较次数较多, 因此它不太适用于小序列.** 同时由于多次任意下标相互交换位置, 相同元素之间原本相对的顺序被破坏了, 因此, 它是不稳定的排序.



### 计数排序（Counting Sort）

计数排序不是基于比较的排序算法，其核心在于将输入的数据值转化为键存储在额外开辟的数组空间中。 作为一种线性时间复杂度的排序，计数排序要求输入的数据必须是有确定范围的整数。

**算法描述**

- 找出待排序的数组中最大和最小的元素
- 统计数组中每个值为i的元素出现的次数，存入数组C的第i项
- 对所有的计数累加（从C中的第一个元素开始，每一项和前一项相加）
- 反向填充目标数组：将每个元素i放在新数组的第C(i)项，每放一个元素就将C(i)减去1

**动图演示**

![计数排序](png/Java/计数排序.gif)

**代码实现**

```java
/**
     * 计数排序算法
     *
     * @param arr 待排序数组
     */
    public static int[] countingSort(int[] arr) {
        // 得到数列的最大值与最小值，并算出差值d
        int max = arr[0];
        int min = arr[0];
        for (int i = 1; i < arr.length; i++) {
            if (arr[i] > max) {
                max = arr[i];
            }
            if (arr[i] < min) {
                min = arr[i];
            }
        }
        int d = max - min;

        // 创建统计数组并计算统计对应元素个数
        int[] countArray = new int[d + 1];
        for (int value : arr) {
            countArray[value - min]++;
        }

        // 统计数组变形，后面的元素等于前面的元素之和
        int sum = 0;
        for (int i = 0; i < countArray.length; i++) {
            sum += countArray[i];
            countArray[i] = sum;
        }

        // 倒序遍历原始数组，从统计数组找到正确位置，输出到结果数组
        int[] sortedArray = new int[arr.length];
        for (int i = arr.length - 1; i >= 0; i--) {
            sortedArray[countArray[arr[i] - min] - 1] = arr[i];
            countArray[arr[i] - min]--;
        }

        return sortedArray;
    }

```

**算法分析**

计数排序是一个稳定的排序算法。当输入的元素是 n 个 0到 k 之间的整数时，时间复杂度是O(n+k)，空间复杂度也是O(n+k)，其排序速度快于任何比较排序算法。当k不是很大并且序列比较集中时，计数排序是一个很有效的排序算法。



### 桶排序（Bucket Sort）

桶排序是计数排序的升级版。它利用了函数的映射关系，高效与否的关键就在于这个映射函数的确定。桶排序 (Bucket sort)的工作的原理：假设输入数据服从均匀分布，将数据分到有限数量的桶里，每个桶再分别排序（有可能再使用别的排序算法或是以递归方式继续使用桶排序进行排）。

![桶排序](png/Java/桶排序.png)

![BucketSort](png/Java/BucketSort.gif)

**算法描述**

- 设置一个定量的数组当作空桶
- 遍历输入数据，并且把数据一个一个放到对应的桶里去
- 对每个不是空的桶进行排序
- 从不是空的桶里把排好序的数据拼接起来

**代码实现**

```java
/**
     * 桶排序算法
     *
     * @param arr 待排序数组
     */
    public static int[] bucketSort(int[] arr) {
        // 计算最大值与最小值
        int max = Integer.MIN_VALUE;
        int min = Integer.MAX_VALUE;
        for (int value : arr) {
            max = Math.max(max, value);
            min = Math.min(min, value);
        }

        // 计算桶的数量
        int bucketNum = (max - min) / arr.length + 1;
        List<List<Integer>> bucketArr = new ArrayList<>(bucketNum);
        for (int i = 0; i < bucketNum; i++) {
            bucketArr.add(new ArrayList<>());
        }

        // 将每个元素放入桶
        for (int value : arr) {
            int num = (value - min) / (arr.length);
            bucketArr.get(num).add(value);
        }

        // 对每个桶进行排序
        for (List<Integer> integers : bucketArr) {
            Collections.sort(integers);
        }

        // 将桶中的元素赋值到原序列
        int index = 0;
        for (List<Integer> integers : bucketArr) {
            for (Integer integer : integers) {
                arr[index++] = integer;
            }
        }

        return arr;
    }
```

**算法分析**

桶排序最好情况下使用线性时间O(n)，桶排序的时间复杂度，取决与对各个桶之间数据进行排序的时间复杂度，因为其它部分的时间复杂度都为O(n)。很显然，桶划分的越小，各个桶之间的数据越少，排序所用的时间也会越少。但相应的空间消耗就会增大。 



# Case

## 双指针

### 删除排序数组中的重复项

给你一个有序数组 nums ，请你 原地 删除重复出现的元素，使每个元素 只出现一次 ，返回删除后数组的新长度。不要使用额外的数组空间，你必须在 原地 修改输入数组 并在使用 O(1) 额外空间的条件下完成。

```java
/**
 * 双指针解决
 */
public int removeDuplicates(int[] A) {
        // 边界条件判断
        if (A == null || A.length == 0){
            return 0;
	}
    
        int left = 0;
        for (int right = 1; right < A.length; right++){
            // 如果左指针和右指针指向的值一样，说明有重复的，
            // 这个时候，左指针不动，右指针继续往右移。如果他俩
            // 指向的值不一样就把右指针指向的值往前挪
            if (A[left] != A[right]){
                A[++left] = A[right];
            }
	}
        return ++left;
}
```



### 移动零

给定一个数组 `nums`，编写一个函数将所有 `0` 移动到数组的末尾，同时保持非零元素的相对顺序。

**示例：**输入: [0,1,0,3,12]，输出: [1,3,12,0,0]

可以参照双指针的思路解决，指针j是一直往后移动的，如果指向的值不等于0才对他进行操作。而i统计的是前面0的个数，我们可以把j-i看做另一个指针，它是指向前面第一个0的位置，然后我们让j指向的值和j-i指向的值交换


```java
public void moveZeroes(int[] nums) {
    int i = 0;// 统计前面0的个数
    for (int j = 0; j < nums.length; j++) {
        if (nums[j] == 0) {//如果当前数字是0就不操作
            i++;
        } else if (i != 0) {
            //否则，把当前数字放到最前面那个0的位置，然后再把
            //当前位置设为0
            nums[j - i] = nums[j];
            nums[j] = 0;
        }
    }
}
```



## 反转

### 旋转数组

给定一个数组，将数组中的元素向右移动 `k` 个位置，其中 `k` 是非负数。

先反转全部数组，在反转前k个，最后在反转剩余的，如下所示：

![旋转数组](png/Java/旋转数组.png)


```java
public void rotate(int[] nums, int k) {
    int length = nums.length;
    k %= length;
    reverse(nums, 0, length - 1);//先反转全部的元素
    reverse(nums, 0, k - 1);//在反转前k个元素
    reverse(nums, k, length - 1);//接着反转剩余的
}

//把数组中从[start，end]之间的元素两两交换,也就是反转
public void reverse(int[] nums, int start, int end) {
    while (start < end) {
        int temp = nums[start];
        nums[start++] = nums[end];
        nums[end--] = temp;
    }
}
```



## 栈

### 判断字符串括号是否合法

字符串中只有字符'('和')'。合法字符串需要括号可以配对。如：()，()()，(())是合法的。)(，()(，(()是非法的。

**① 栈**

![判断字符串括号是否合法](png/Java/判断字符串括号是否合法.gif)

```java
/**
 * 判断字符串括号是否合法
 *
 * @param s
 * @return
 */
public boolean isValid(String s) {
        // 当字符串本来就是空的时候，我们可以快速返回true
        if (s == null || s.length() == 0) {
            return true;
        }

        // 当字符串长度为奇数的时候，不可能是一个有效的合法字符串
        if (s.length() % 2 == 1) {
            return false;
        }

        // 消除法的主要核心逻辑:
        Stack<Character> t = new Stack<>();
        for (int i = 0; i < s.length(); i++) {
            // 取出字符
            char c = s.charAt(i);
            if (c == '(') {
                // 如果是'('，那么压栈
                t.push(c);
            } else if (c == ')') {
                // 如果是')'，那么就尝试弹栈
                if (t.empty()) {
                    // 如果弹栈失败，那么返回false
                    return false;
                }
                t.pop();
            }
        }

        return t.empty();
}
```

**复杂度分析**：每个字符只入栈一次，出栈一次，所以时间复杂度为 O(N)，而空间复杂度为 O(N)，因为最差情况下可能会把整个字符串都入栈。



**② 栈深度扩展**

如果仔细观察，你会发现，栈中存放的元素是一样的。全部都是左括号'('，除此之外，再也没有别的元素，优化方法如下。**栈中元素都相同时，实际上没有必要使用栈，只需要记录栈中元素个数。** 我们可以通过画图来解决这个问题，如下图所示：

![leftBraceNumber加减](png/Java/leftBraceNumber加减.gif)

```java
/**
 * 判断字符串括号是否合法
 *
 * @param s
 * @return
 */
public boolean isValid(String s) {
        // 当字符串本来就是空的时候，我们可以快速返回true
        if (s == null || s.length() == 0) {
            return true;
        }

        // 当字符串长度为奇数的时候，不可能是一个有效的合法字符串
        if (s.length() % 2 == 1) {
            return false;
        }

        // 消除法的主要核心逻辑:
        int leftBraceNumber = 0;
        for (int i = 0; i < s.length(); i++) {
            // 取出字符
            char c = s.charAt(i);
            if (c == '(') {
                // 如果是'('，那么压栈
                leftBraceNumber++;
            } else if (c == ')') {
                // 如果是')'，那么就尝试弹栈
                if (leftBraceNumber <= 0) {
                    // 如果弹栈失败，那么返回false
                    return false;
                }
                --leftBraceNumber;
            }
        }

        return leftBraceNumber == 0;
}
```

**复杂度分析**：每个字符只入栈一次，出栈一次，所以时间复杂度为 O(N)，而空间复杂度为 O(1)，因为我们已经只用一个变量来记录栈中的内容。



**③ 栈广度扩展**

接下来再来看看如何进行广度扩展。观察题目可以发现，栈中只存放了一个维度的信息：左括号'('和右括号')'。如果栈中的内容变得更加丰富一点，就可以得到下面这道扩展题。给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串，判断字符串是否有效。有效字符串需满足：

- 左括号必须用相同类型的右括号闭合
- 左括号必须以正确的顺序闭合
- 注意空字符串可被认为是有效字符串

```java
/**
 * 判断字符串括号是否合法
 *
 * @param s
 * @return
 */
public boolean isValid(String s) {
        if (s == null || s.length() == 0) {
            return true;
        }
        if (s.length() % 2 == 1) {
            return false;
        }

        Stack<Character> t = new Stack<>();
        for (int i = 0; i < s.length(); i++) {
            char c = s.charAt(i);
            if (c == '{' || c == '(' || c == '[') {
                t.push(c);
            } else if (c == '}') {
                if (t.empty() || t.peek() != '{') {
                    return false;
                }
                t.pop();
            } else if (c == ']') {
                if (t.empty() || t.peek() != '[') {
                    return false;
                }
                t.pop();
            } else if (c == ')') {
                if (t.empty() || t.peek() != '(') {
                    return false;
                }
                t.pop();
            } else {
                return false;
            }
        }

        return t.empty();
}
```



### 大鱼吃小鱼

在水中有许多鱼，可以认为这些鱼停放在 x 轴上。再给定两个数组 Size，Dir，Size[i] 表示第 i 条鱼的大小，Dir[i] 表示鱼的方向 （0 表示向左游，1 表示向右游）。这两个数组分别表示鱼的大小和游动的方向，并且两个数组的长度相等。鱼的行为符合以下几个条件:

- 所有的鱼都同时开始游动，每次按照鱼的方向，都游动一个单位距离
- 当方向相对时，大鱼会吃掉小鱼；
- 鱼的大小都不一样。

输入：Size = [4, 3, 2, 1, 5], Dir = [1, 1, 1, 1, 0]，输出：1。请完成以下接口来计算还剩下几条鱼？

![大鱼吃小鱼](png/Java/大鱼吃小鱼.gif)

```java
/**
 * 大鱼吃小鱼
 *
 * @param fishSize
 * @param fishDirection
 * @return
 */
public int solution(int[] fishSize, int[] fishDirection) {
        // 首先拿到鱼的数量: 如果鱼的数量小于等于1，那么直接返回鱼的数量
        final int fishNumber = fishSize.length;
        if (fishNumber <= 1) {
            return fishNumber;
        }

        // 0表示鱼向左游
        final int left = 0;
        // 1表示鱼向右游
        final int right = 1;
        Stack<Integer> t = new Stack<>();
        for (int i = 0; i < fishNumber; i++) {
            // 当前鱼的情况：1，游动的方向；2，大小
            final int curFishDirection = fishDirection[i];
            final int curFishSize = fishSize[i];
            // 当前的鱼是否被栈中的鱼吃掉了
            boolean hasEat = false;
            // 如果栈中还有鱼，并且栈中鱼向右，当前的鱼向左游，那么就会有相遇的可能性
            while (!t.empty() && fishDirection[t.peek()] == right && curFishDirection == left) {
                // 如果栈顶的鱼比较大，那么把新来的吃掉
                if (fishSize[t.peek()] > curFishSize) {
                    hasEat = true;
                    break;
                }
                // 如果栈中的鱼较小，那么会把栈中的鱼吃掉，栈中的鱼被消除，所以需要弹栈。
                t.pop();
            }
            // 如果新来的鱼，没有被吃掉，那么压入栈中。
            if (!hasEat) {
                t.push(i);
            }
        }

        return t.size();
}
```



### 找出数组中右边比我小的元素

一个整数数组 A，找到每个元素：右边第一个比我小的下标位置，没有则用 -1 表示。输入：[5, 2]，输出：[1, -1]。

```java
/**
 * 找出数组中右边比我小的元素
 *
 * @param A
 * @return
 */
public static int[] findRightSmall(int[] A) {
        // 结果数组
        int[] ans = new int[A.length];
        // 注意，栈中的元素记录的是下标
        Stack<Integer> t = new Stack<>();
        for (int i = 0; i < A.length; i++) {
            final int x = A[i];
            // 每个元素都向左遍历栈中的元素完成消除动作
            while (!t.empty() && A[t.peek()] > x) {
                // 消除的时候，记录一下被谁消除了
                ans[t.peek()] = i;
                // 消除时候，值更大的需要从栈中消失
                t.pop();
            }
            // 剩下的入栈
            t.push(i);
        }
        // 栈中剩下的元素，由于没有人能消除他们，因此，只能将结果设置为-1。
        while (!t.empty()) {
            ans[t.peek()] = -1;
            t.pop();
        }
        return ans;
}
```



### 字典序最小的 k 个数的子序列

给定一个正整数数组和 k，要求依次取出 k 个数，输出其中数组的一个子序列，需要满足：1. **长度为 k**；2.**字典序最小**。

输入：nums = [3,5,2,6], k = 2，输出：[2,6]

**解释**：在所有可能的解：{[3,5], [3,2], [3,6], [5,2], [5,6], [2,6]} 中，[2,6] 字典序最小。所谓字典序就是，给定两个数组：x = [x1,x2,x3,x4]，y = [y1,y2,y3,y4]，如果 0 ≤ p < i，xp == yp 且 xi < yi，那么我们认为 x 的字典序小于 y。

```java
/**
 * 字典序最小的 k 个数的子序列
 *
 * @param nums
 * @param k
 * @return
 */
public int[] findSmallSeq(int[] nums, int k) {
        int[] ans = new int[k];
        Stack<Integer> s = new Stack();
        // 这里生成单调栈
        for (int i = 0; i < nums.length; i++) {
            final int x = nums[i];
            final int left = nums.length - i;
            // 注意我们想要提取出k个数，所以注意控制扔掉的数的个数
            while (!s.empty() && (s.size() + left > k) && s.peek() > x) {
                s.pop();
            }
            s.push(x);
        }
        // 如果递增栈里面的数太多，那么我们只需要取出前k个就可以了。
        // 多余的栈中的元素需要扔掉。
        while (s.size() > k) {
            s.pop();
        }
        // 把k个元素取出来，注意这里取的顺序!
        for (int i = k - 1; i >= 0; i--) {
            ans[i] = s.peek();
            s.pop();
        }
        return ans;
}
```



## FIFO队列

### 二叉树的层次遍历（两种方法）

从上到下按层打印二叉树，同一层结点按从左到右的顺序打印，每一层打印到一行。输入：

![二叉树的层次遍历](png/Java/二叉树的层次遍历.png)

输出：[[3], [9, 8], [6, 7]]

```java
// 二叉树结点的定义
public class TreeNode {
      // 树结点中的元素值
      int val = 0;
      // 二叉树结点的左子结点
      TreeNode left = null;
      // 二叉树结点的右子结点
      TreeNode right = null;
}
```

**Queue表示FIFO队列解法**：

![Queue表示FIFO队列解法](png/Java/Queue表示FIFO队列解法.gif)

```java
public List<List<Integer>> levelOrder(TreeNode root) {
        // 生成FIFO队列
        Queue<TreeNode> Q = new LinkedList<>();
        // 如果结点不为空，那么加入FIFO队列
        if (root != null) {
               Q.offer(root);
        }

        // ans用于保存层次遍历的结果
        List<List<Integer>> ans = new LinkedList<>();
        // 开始利用FIFO队列进行层次遍历
        while (Q.size() > 0) {
                // 取出当前层里面元素的个数
                final int qSize = Q.size();
                // 当前层的结果存放于tmp链表中
                List<Integer> tmp = new LinkedList<>();
                // 遍历当前层的每个结点
                for (int i = 0; i < qSize; i++) {
                    // 当前层前面的结点先出队
                    TreeNode cur = Q.poll();
                    // 把结果存放当于当前层中
                    tmp.add(cur.val);
                    // 把下一层的结点入队，注意入队时需要非空才可以入队。
                    if (cur.left != null) {
                        Q.offer(cur.left);
                    }
                    if (cur.right != null) {
                        Q.offer(cur.right);
                    }
                }

                // 把当前层的结果放到返回值里面。
                ans.add(tmp);
        }

        return ans;
}
```

**ArrayList表示FIFO队列解法**：

![ArrayList表示FIFO队列解法](png/Java/ArrayList表示FIFO队列解法.gif)

```java
public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> ans = new ArrayList<>();
        // 初始化当前层结点
        List<TreeNode> curLevel = new ArrayList<>();
        // 注意：需要root不空的时候才加到里面。
        if (root != null) {
                curLevel.add(root);
        }
        while (curLevel.size() > 0) {
                // 准备用来存放下一层的结点
                List<TreeNode> nextLevel = new ArrayList<>();
                // 用来存放当前层的结果
                List<Integer> curResult = new ArrayList<>();
                // 遍历当前层的每个结点
                for (TreeNode cur : curLevel) {
                    // 把当前层的值存放到当前结果里面
                    curResult.add(cur.val);
                    // 生成下一层
                    if (cur.left != null) {
                        nextLevel.add(cur.left);
                    }
                    if (cur.right != null) {
                        nextLevel.add(cur.right);
                    }
                }
                // 注意这里的更迭!滚动前进
                curLevel = nextLevel;
                // 把当前层的值放到结果里面
                ans.add(curResult);
        }
        return ans;
}
```



### 循环队列

设计一个可以容纳 k 个元素的循环队列。需要实现以下接口：

```java
public class MyCircularQueue {
        // 参数k表示这个循环队列最多只能容纳k个元素
        public MyCircularQueue(int k){}
        // 将value放到队列中, 成功返回true
        public boolean enQueue(int value){return false;}
        // 删除队首元素，成功返回true
        public boolean deQueue(){return false;}
        // 得到队首元素，如果为空，返回-1
        public int Front(){return 0;}
        // 得到队尾元素，如果队列为空，返回-1
        public int Rear(){return 0;}
        // 看一下循环队列是否为空
        public boolean isEmpty(){return false;}
        // 看一下循环队列是否已放满k个元素
        public boolean isFull(){return false;}
}
```

循环队列的重点在于**循环使用固定空间**，难点在于**控制好 front/rear 两个首尾指示器**。

**方法一：k个元素空间**

只使用 k 个元素的空间，三个变量 front, rear, used 来控制循环队列。标记 k = 6 时，循环队列的三种情况，如下图所示：

![循环队列k个元素空间](png/Java/循环队列k个元素空间.png)

```java
public class MyCircularQueue {
        // 已经使用的元素个数
        private int used = 0;
        // 第一个元素所在位置
        private int front = 0;
        // rear是enQueue可在存放的位置，注意开闭原则， [front, rear)
        private int rear = 0;
        // 循环队列最多可以存放的元素个数
        private int capacity = 0;
        // 循环队列的存储空间
        private int[] a = null;

        public MyCircularQueue(int k) {
            // 初始化循环队列
            capacity = k;
            a = new int[capacity];
        }

        public boolean enQueue(int value) {
            // 如果已经放满了
            if (isFull()) {
                return false;
            }

            // 如果没有放满，那么a[rear]用来存放新进来的元素
            a[rear] = value;
            // rear注意取模
            rear = (rear + 1) % capacity;
            // 已经使用的空间
            used++;
            // 存放成功!
            return true;
        }

        public boolean deQueue() {
            // 如果是一个空队列，当然不能出队
            if (isEmpty()) {
                return false;
            }

            // 第一个元素取出
            int ret = a[front];
            // 注意取模
            front = (front + 1) % capacity;
            // 已经存放的元素减减
            used--;
            // 取出元素成功
            return true;
        }

        public int front() {
            // 如果为空，不能取出队首元素
            if (isEmpty()) {
                return -1;
            }

            // 取出队首元素
            return a[front];
        }

        public int rear() {
            // 如果为空，不能取出队尾元素
            if (isEmpty()) {
                return -1;
            }

            // 注意：这里不能使用rear - 1
            // 需要取模
            int tail = (rear - 1 + capacity) % capacity;
            return a[tail];
        }

        // 队列是否为空
        public boolean isEmpty() {
            return used == 0;
        }

        // 队列是否满了
        public boolean isFull() {
            return used == capacity;
        }
}
```

**复杂度分析**：入队操作与出队操作都是 O(1)。

**方法二：k+1个元素空间**

方法 1 利用 used 变量对满队列和空队列进行了区分。实际上，这种区分方式还有另外一种办法，使用 k+1 个元素的空间，两个变量 front, rear 来控制循环队列的使用。具体如下：

- 在申请数组空间的时候，申请 k + 1 个空间
- 在放满循环队列的时候，必须要保证 rear 与 front 之间有空隙

如下图（此时 k = 5）所示：

![循环队列k+1个元素空间](png/Java/循环队列k+1个元素空间.png)

```java
public class MyCircularQueue {
        // 队列的头部元素所在位置
        private int front = 0;
        // 队列的尾巴，注意我们采用的是前开后闭原则， [front, rear)
        private int rear = 0;
        private int[] a = null;
        private int capacity = 0;

        public MyCircularQueue(int k) {
            // 初始化队列，注意此时队列中元素个数为
            // k + 1
            capacity = k + 1;
            a = new int[k + 1];
        }

        public boolean enQueue(int value) {
            // 如果已经满了，无法入队
            if (isFull()) {
                return false;
            }
            // 把元素放到rear位置
            a[rear] = value;
            // rear向后移动
            rear = (rear + 1) % capacity;
            return true;
        }

        public boolean deQueue() {
            // 如果为空，无法出队
            if (isEmpty()) {
                return false;
            }
            // 出队之后，front要向前移
            front = (front + 1) % capacity;
            return true;
        }

        public int front() {
            // 如果能取出第一个元素，取a[front];
            return isEmpty() ? -1 : a[front];
        }

        public int rear() {
            // 由于我们使用的是前开后闭原则
            // [front, rear)
            // 所以在取最后一个元素时，应该是
            // (rear - 1 + capacity) % capacity;
            int tail = (rear - 1 + capacity) % capacity;
            return isEmpty() ? -1 : a[tail];
        }

        public boolean isEmpty() {
            // 队列是否为空
            return front == rear;
        }

        public boolean isFull() {
            // rear与front之间至少有一个空格
            // 当rear指向这个最后的一个空格时，
            // 队列就已经放满了!
            return (rear + 1) % capacity == front;
        }
}
```

**复杂度分析**：入队与出队操作都是 O(1)。



## 单调队列

单调队列属于**双端队列**的一种。双端队列与 FIFO 队列的区别在于：

- FIFO 队列只能从尾部添加元素，首部弹出元素
- 双端队列可以从首尾两端 push/pop 元素

### 滑动窗口的最大值

给定一个数组和滑动窗口的大小，请找出所有滑动窗口里的最大值。输入：nums = [1,3,-1,-3,5,3], k = 3，输出：[3,3,5,5]。

![滑动窗口的最大值](png/Java/滑动窗口的最大值.png)

```java
public class Solution {

    // 单调队列使用双端队列来实现
    private ArrayDeque<Integer> Q = new ArrayDeque<>();

    // 入队的时候，last方向入队，但是入队的时候
    // 需要保证整个队列的数值是单调的
    // (在这个题里面我们需要是递减的)
    // 并且需要注意，这里是Q.getLast() < val
    // 如果写成Q.getLast() <= val就变成了严格单调递增
    private void push(int val) {
        while (!Q.isEmpty() && Q.getLast() < val) {
            Q.removeLast();
        }
        // 将元素入队
        Q.addLast(val);
    }

    // 出队的时候，要相等的时候才会出队
    private void pop(int val) {
        if (!Q.isEmpty() && Q.getFirst() == val) {
            Q.removeFirst();
        }
    }

    public int[] maxSlidingWindow(int[] nums, int k) {
        List<Integer> ans = new ArrayList<>();
        for (int i = 0; i < nums.length; i++) {
            push(nums[i]);
            // 如果队列中的元素还少于k个
            // 那么这个时候，还不能去取最大值
            if (i < k - 1) {
                continue;
            }
            // 队首元素就是最大值
            ans.add(Q.getFirst());
            // 尝试去移除元素
            pop(nums[i - k + 1]);
        }
        // 将ans转换成为数组返回!
        return ans.stream().mapToInt(Integer::valueOf).toArray();
    }

}
```

**复杂度分析**：每个元素都只入队一次，出队一次，每次入队与出队都是 O(1) 的复杂度，因此整个算法的复杂度为 O(n)。



### 捡金币游戏

给定一个数组 A[]，每个位置 i 放置了金币 A[i]，小明从 A[0] 出发。当小明走到 A[i] 的时候，下一步他可以选择 A[i+1, i+k]（当然，不能超出数组边界）。每个位置一旦被选择，将会把那个位置的金币收走（如果为负数，就要交出金币）。请问，最多能收集多少金币？

输入：[1,-1,-100,-1000,100,3], k = 2，输出：4。

**解释**：从 A[0] = 1 出发，收获金币 1。下一步走往 A[2] = -100, 收获金币 -100。再下一步走到 A[4] = 100，收获金币 100，最后走到 A[5] = 3，收获金币 3。最多收获 1 - 100 + 100 + 3 = 4。没有比这个更好的走法了。

```java
public class Solution {
    public int maxResult(int[] A, int k) {
        // 处理掉各种边界条件!
        if (A == null || A.length == 0 || k <= 0) {
            return 0;
        }
        final int N = A.length;
        // 每个位置可以收集到的金币数目
        int[] get = new int[N];
        // 单调队列，这里并不是严格递减
        ArrayDeque<Integer> Q = new ArrayDeque<Integer>();
        for (int i = 0; i < N; i++) {
            // 在取最大值之前，需要保证单调队列中都是有效值。
            // 也就是都在区间里面的值
            // 当要求get[i]的时候，
            // 单调队列中应该是只能保存[i-k, i-1]这个范围
            if (i - k > 0) {
                if (!Q.isEmpty() && Q.getFirst() == get[i - k - 1]) {
                    Q.removeFirst();
                }
            }
            // 从单调队列中取得较大值
            int old = Q.isEmpty() ? 0 : Q.getFirst();
            get[i] = old + A[i];
            // 入队的时候，采用单调队列入队
            while (!Q.isEmpty() && Q.getLast() < get[i]) {
                Q.removeLast();
            }
            Q.addLast(get[i]);
        }
        return get[N - 1];
    }
    
}
```

**复杂度分析**：每个元素只入队一次，出队一次，每次入队与出队复杂度都是 O(n)。因此，时间复杂度为 O(n)，空间复杂度为 O(n)。

# 参考资料

参考书籍：《Java数据结构和算法》

参考资料：动画排序http://www.donghuasuanfa.com/sort

十大经典排序算法动画与解析

算法思想：https://www.cnblogs.com/steven_oyj/archive/2010/05/22/1741378.html