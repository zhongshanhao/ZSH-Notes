## vector

变长数组

定义

```c++
vector<int> a;				// 一维变长的数组
vector<int > graph[10];	// 一维定长的数组，每个数组元素都是变长的数组
vector<vector<int> > graph;  // 两维都是变长的数组
```

push_back()/pop_back()/迭代

```c++
vector<int> a;
// 在末尾添加一个元素
a.push_back(1);
a.push_back(2);
a.push_back(3);
// 移除最后一个元素
a.pop_back();
// 遍历
for(int i = 0; i < a.size(); i++)
    cout<< a[i] << endl;
```

size()--获取vector元素个数

clear()--清空vector中所有元素

insert(it, x)--用来在vector的任意迭代器it处插入一个元素x

```c++
a.insert(a.begin()+1, 5); // 在位置1处插入元素5，将其后的元素依次往后移动一位
```

erase(it) -- 用来删除迭代器为it处的元素

erase(first, last) -- 用来删除[first, last)内的所有元素，first与last均为迭代器

图的存储

```c++
struct Node{
    int v;
    int w;
    Node(int _v, int _w): v(_v), w(_w) { }
}

// 定义邻接表
vector<Node> graph[N];
// 加入一条边0->3，权重为4
graph[0].push_back(Node(3, 4));
```

## Set

集合，是一个内部自动有序且不含重复元素的容器

insert(x) -- 将x插入set容器，并自动递增排序和去重

find(value) -- 返回set中对应值为value的迭代器

erase(it) -- 删除迭代器it所指向的元素

erase(value) -- 删除set中值为value的元素

erase(first, last) -- 可以删除一个区间内的所有元素

size() -- 用来获得set内元素的个数

clear() -- 用来清空set中所有元素

```c++
set<int> s;
s.insert(3);
s.insert(2);
s.insert(4);
// 遍历
for(auto &i : s)
    cout<< i << " " << endl;
set<int>::iterator it = s.find(68);
cout<< *it << endl;
s.erase(it);
```

## String

```c++
int main(){
    string str = "abcd";
    // 迭代方式一
    for(int i = 0; i < str.length(); i++)
        cout<< str[i] << " ";
    // 迭代方式二
    for(auto &c : str)
        cout<< c << " ";
    cout<< str;
    printf("%s", str.c_str());
}
```

字符串拼接 +=

字典序比较 <，>，!=

insert(pos, string) -- 在pos号位置插入字符串string

insert(it, it2, it3) -- it为原字符串的欲插入位置，it2和it3为待插入字符串的首尾迭代器，用来表示串[it2, it3)将被插在it的位置上

erase(it) -- 删除迭代器it处的元素

erase(first, last) -- 删除区间[first, last)内的元素

erase(pos, length) -- 从pos为需要开始删除的起始位置，length为删除的字符个数

clear() -- 清空string中的数据

substr(pos, len) -- 返回从pos号位开始、长度为len的子串

find() -- str.find(str2)，当str2是str的子串时，返回其在str中第一次出现的位置，如果str2不是str的子串，那么返回string::npos

replace()

str.replace(pos, len, str2) 把str从pos号位开始、长度为len的子串替换为str2

str.replace(it1, it2, str2) 把str的迭代器[it1, it2)范围的子串替换为str2

## map

```c++
map<char, int> m;
m['a'] = 0;
m['c'] = 20;
printf("%d", m['a']);

// map的遍历
for(auto &it : m){
    printf("key: %c value: %d", it.first, it.second);
    cout<<endl;
}

// 判断key是否在map中，如果不在则返回m.end()
if(m.find('b') == m.end())
    cout<< "not exist";

// 删除key为'a'的键
m.erase('a');
```

## queue

先进先出的容器

push()入队

front()、back()分别获得队首元素和队尾元素

pop() 令队首元素出队，注意返回值为空

empty() 判断队列是否为空

size() 获得队列元素个数

## stack

push() 将元素进栈

top() 获取栈顶元素

pop() 弹出栈顶元素

empty() 判断栈是否为空

size() 获取栈内元素个数

## pair

```c++
#include<map> //加此头文件

int main(){
    pair<int, int> p(1, 2);
    printf("%d %d", p.first, p.second);
}
```

给TreeNode分配空间，并返回指向该内存空间的指针

```c++
TreeNode* root = new TreeNode(0);
```
