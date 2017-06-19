---
title: AVL树的平衡算法（JAVA实现）
date: 2017-06-19 14:29:39
tags: 算法
---

## 概念
　　AVL树本质上还是一个二叉搜索树，不过比二叉搜索树多了一个平衡条件：每个节点的左右子树的高度差不大于1。二叉树的应用是为了弥补链表的查询效率问题，但是极端情况下，二叉搜索树会无限接近于链表，这种时候就无法体现二叉搜索树在查询时的高效率，而最初出现的解决方式就是AVL树。如下图：
<!-- more -->
![](/images/mybatis_30.png)

## 旋转
　　说到AVL树就不得不提到树的旋转，旋转是AVL维持平衡的方式，主要有以下四种类型。
### 左左旋转
　　如图2-1所示，此时A节点的左树与右树的高度差为2，不符合AVL的定义，此时以B节点为轴心，AB间连线为转轴，将A节点旋转至B节点下方，由B节点的父节点变成子节点。实现所有节点的左右子树高度差小于等于1。如图2-2。
![](/images/mybatis_31.png)

### 右右旋转
　　右右旋转与左左旋转类似，但是动作相反，如图2-3，2-4所示，以B节点为轴心，BC间连线为轴，将C节点旋转至B下方，成为B的子节点。实现所有节点的左右子树高度差小于等于1。
![](/images/mybatis_32.png)

### 左右旋转
　　左右旋转稍复杂一点，需要旋转两次，如果用左左旋转的方式直接旋转图2-5中的树，会变成图2-8的样子，此时B节点的左右子树高度差依然没有平衡，所以要先对2-5的树做一步处理，就是以BC为轴，将C节点旋转至B节点的位置，如图2-6所示，此时就将树转换成了左左旋转的场景，之后使用左左旋转即可完成平衡。
![](/images/mybatis_33.png)

### 右左旋转
　　右左旋转与左右旋转类似，也是旋转两次。如下图，具体细节请看下面的代码详解。
![](/images/mybatis_34.png)

## 代码实现
### 新增
　　二叉树的插入不在赘述，这里只分析插入时的平衡方法。如果新增节点没有兄弟节点时会引起树的高度变化，此时需要对上层节点的平衡值进行修改，如果出现了不平衡树，则需要调用平衡方法，代码如下：

	private void balance(Node node){
        Node parent = node.getParent();
        Node node_middle = node;
        Node node_prev = node;
        
        Boolean avl = true;
        do{
            if(node_middle == parent.getLeft() && (-1 <= parent.getAVL()-1 && parent.getAVL()-1 <= 1)){
                //node_middle为parent的左树，此时parent左树高度+1不会造成不平衡。
                parent.subAVL();
                node_prev = node_middle;
                node_middle = parent;
                //由于上面对parent的平衡值进行了修改，如果修改后的平衡值为0，说明此时parent节点的高度没有改变，之前较短的左树高度+1，变为与右树高度相同。
                if(parent != null && parent.getAVL() == 0)
                    parent = null;
                else
                    parent = parent.getParent();
            }else if(node_middle == parent.getRight() && (-1 <= parent.getAVL()+1 && parent.getAVL()+1 <= 1)){
                //node_middle为parent的右树，此时parent右树高度+1不会造成不平衡。
                parent.addAVL();
                node_prev = node_middle;
                node_middle = parent;
                //由于上面对parent的平衡值进行了修改，如果修改后的平衡值为0，说明此时parent节点的高度没有改变，之前较短的右树高度+1，变为与左树高度相同。
                if(parent != null && parent.getAVL() == 0)
                    parent = null;
                else
                    parent = parent.getParent();
            }else{//出现最小不平衡节点，新增时不需要考虑更高节点，所以直接中断循环，调用平衡方法
                avl = false;
            }
        }while(parent != null && avl);
        
        if(parent == null){
            return;
        }
        //选择相应的旋转方式
        chooseCalculation(parent, node_middle, node_prev);
    }

### 删除
　　删除较新增复杂一些，主要是因为存在一次旋转无法达到平衡效果的情况，而且删除本身也分为三种情况，分别是：
#### 删除叶子节点
　　由于叶子节点没有子树，不涉及替换的问题，所以直接删除即可，如果删除节点没有兄弟节点会引起高度变化，此时依次对父级节点的平衡值做对应修改，如果出现不平衡树则要进行旋转。
#### 删除节点存在一个子节点
　　此时将子节点上移，替换删除节点的位置，这个操作势必会造成所在子树的高度变化，所以需要依次对父级节点的平衡值做对应修该，如果出现不平衡树进行旋转操作。
#### 删除节点存在两个子节点
　　这种情况比较复杂，由于存在两个子节点，所以不能简单的将其中一个子节点提高一级，因此需要在节点的子树中搜索一个合适的节点进行替换，之前自己构思的时候选择的是搜寻一个最长子树的最接近删除节点的值的叶子节点，具体实现的时候发现需要考虑的场景太多，并不适合，参考算法导论上的思路后改成了搜寻左树的最大节点，此时只需考虑左树的情况即可。实现方法如下：

	public void deleteNode(int item){

        Node node = get(item);
        if(node == null)
            return;
        Node parent = node.getParent();
        if(!node.hasChild()){												//叶子节点
            if(parent == null){												//删除最后节点
                root = null;
                return;
            }
            if(node.hasBrother()){											//node有兄弟节点时，需要判断是否需要调用平衡方法
                if(node == parent.getLeft())
                    isBalance(node, 1);
                else
                    isBalance(node, -1);
                parent.deleteChildNode(node);
            }else{															//node没有兄弟节点时，高度减一，需要进行平衡
                deleteAvl(node);
                parent.deleteChildNode(node);
            }
        }else if(node.getLeft() != null && node.getRight() == null){		//有一个子节点时，将子节点上移一位，然后进行平衡即可
            if(parent == null){												//删除的是根节点
                root = node;
                return;
            }
            if(node == parent.getLeft()){
                parent.setLeft(node.getLeft());
            }else{
                parent.setRight(node.getLeft());
            }
            node.getLeft().setParent(parent);
            deleteAvl(node.getLeft());
        }else if(node.getLeft() == null && node.getRight() != null){		//有一个子节点时，将子节点上移一位，然后进行平衡即可
            if(parent == null){												//删除的是根节点
                root = node;
                return;
            }
            if(node == parent.getRight()){
                parent.setRight(node.getRight());
            }else{
                parent.setLeft(node.getRight());
            }
            node.getRight().setParent(parent);
            deleteAvl(node.getRight());
        }
        else{																//有两个子节点时，先在节点左树寻找最大节点last，然后删除last，最后将被删除节点的value替换为last的value
            Node last = findLastNode(node);
            int tmp = last.getValue();
            deleteNode(last.getValue());
            node.setValue(tmp);
        }
        node = null;														//GC
    }

### 旋转
#### 左左旋转

	private void LeftLeftRotate(Node node){
        
        Node parent = node.getParent();
        
        if(parent.getParent() != null && parent == parent.getParent().getLeft()){
            node.setParent(parent.getParent());
            parent.getParent().setLeft(node);
        }else if(parent.getParent() != null && parent == parent.getParent().getRight()){
            node.setParent(parent.getParent());
            parent.getParent().setRight(node);
        }else{
            root = node;
            node.setParent(null);
        }
        parent.setParent(node);
        parent.setLeft(node.getRight());
        if(node.getRight() != null)
            node.getRight().setParent(parent);
        node.setRight(parent);
        
        if(node.getAVL() == -1){										//只有左节点时，parent转换后没有子节点
            parent.setAVL(0);
            node.setAVL(0);
        }else if(node.getAVL() == 0){									//node有两个子节点，转换后parent有一个左节点
            parent.setAVL(-1);
            node.setAVL(1);
        }																//node.getAVL()为1时会调用左右旋转
    }

#### 右右旋转

	private void RightRightRotate(Node node){
        
        Node parent = node.getParent();
        
        if(parent.getParent() != null && parent == parent.getParent().getLeft()){
            node.setParent(parent.getParent());
            parent.getParent().setLeft(node);
        }else if(parent.getParent() != null && parent == parent.getParent().getRight()){
            node.setParent(parent.getParent());
            parent.getParent().setRight(node);
        }else{
            root = node;
            node.setParent(null);
        }
        parent.setParent(node);
        parent.setRight(node.getLeft());
        if(node.getLeft() != null)
            node.getLeft().setParent(parent);
        node.setLeft(parent);
        
        if(node.getAVL() == 1){
            node.setAVL(0);
            parent.setAVL(0);
        }else if(node.getAVL() == 0){									//当node有两个节点时，转换后层数不会更改，左树比右树高1层，parent的右树比左树高一层
            parent.setAVL(1);
            node.setAVL(-1);
        }
    }

#### 左右旋转

	private void LeftRightRotate(Node node){
        
        Node parent = node.getParent();
        Node child = node.getRight();
        
        //左右旋转时node的avl必为1，所以只需考虑child的avl
        if(!child.hasChild()){
            node.setAVL(0);
            parent.setAVL(0);
        }else if(child.getAVL() == -1){
            node.setAVL(0);
            parent.setAVL(1);
        }else if(child.getAVL() == 1){
            node.setAVL(-1);
            parent.setAVL(0);
        }else if(child.getAVL() == 0){
            node.setAVL(0);
            parent.setAVL(0);
        }
        child.setAVL(0);
        
        //第一次交换
        parent.setLeft(child);
        node.setParent(child);
        node.setRight(child.getLeft());
        if(child.getLeft() != null)
            child.getLeft().setParent(node);
        child.setLeft(node);
        child.setParent(parent);
        
        //第二次交换
        if(parent.getParent() != null && parent == parent.getParent().getLeft()){
            child.setParent(parent.getParent());
            parent.getParent().setLeft(child);
        }else if(parent.getParent() != null && parent == parent.getParent().getRight()){
            child.setParent(parent.getParent());
            parent.getParent().setRight(child);
        }else{
            root = child;
            child.setParent(null);
        }
        parent.setParent(child);
        parent.setLeft(child.getRight());
        if(child.getRight() != null)
            child.getRight().setParent(parent);
        child.setRight(parent);
    }

#### 右左旋转

	private void RightLeftRotate(Node node){
        
        Node parent = node.getParent();
        Node child = node.getLeft();
        
        if(!child.hasChild()){
            node.setAVL(0);
            parent.setAVL(0);
        }else if(child.getAVL() == -1){
            node.setAVL(1);
            parent.setAVL(0);
        }else if(child.getAVL() == 1){
            node.setAVL(0);
            parent.setAVL(-1);
        }else if(child.getAVL() == 0){
            parent.setAVL(0);
            node.setAVL(0);
        }
        child.setAVL(0);
        
        //第一次交换
        parent.setRight(child);
        node.setParent(child);
        node.setLeft(child.getRight()); 
        if(child.getRight() != null)
            child.getRight().setParent(node);
        child.setRight(node);
        child.setParent(parent);
        
        //第二次交换
        if(parent.getParent() != null && parent == parent.getParent().getLeft()){
            child.setParent(parent.getParent());
            parent.getParent().setLeft(child);
        }else if(parent.getParent() != null && parent == parent.getParent().getRight()){
            child.setParent(parent.getParent());
            parent.getParent().setRight(child);
        }else{
            root = child;
            child.setParent(null);
        }
        parent.setParent(child);
        parent.setRight(child.getLeft());
        if(child.getLeft() != null)
            child.getLeft().setParent(parent);
        child.setLeft(parent);
    }

完整代码github地址：https://github.com/ziyuanjg/AVLTree