---
title: 【HDU5831】Rikka with Parenthesis II
date: 2020-06-13 12:24:09
tags:
---

### 题意

> As we know, Rikka is poor at math. Yuta is worrying about this situation, so he gives Rikka some math tasks to practice. There is one of them:
> 
> Correct parentheses sequences can be defined recursively as follows:
> 1.The empty string "" is a correct sequence.
> 2.If "X" and "Y" are correct sequences, then "XY" (the concatenation of X and Y) is a correct sequence.
> 3.If "X" is a correct sequence, then "(X)" is a correct sequence.
> Each correct parentheses sequence can be derived using the above rules.
> Examples of correct parentheses sequences include "", "()", "()()()", "(()())", and "(((())))".
> 
> Now Yuta has a parentheses sequence S, and he wants Rikka to choose two different position i,j and swap Si,Sj.
> 
> Rikka likes correct parentheses sequence. So she wants to know if she can change S to a correct parentheses sequence after this operation.
> 
> It is too difficult for Rikka. Can you help her?

### 输入

> The first line contains a number t(1<=t<=1000), the number of the testcases. And there are no more then 10 testcases with n>100

> For each testcase, the first line contains an integers n(1<=n<=100000), the length of S. And the second line contains a string of length S which only contains ‘(’ and ‘)’.

### 输出

> For each testcase, print "Yes" or "No" in a line.

### 样例输入

    3
	4
	())(
	4
	()()
	6
	)))(((

### 样例输出

	Yes
	Yes
	No

### 提示

> For the second sample input, Rikka can choose (1,3) or (2,4) to swap. But do nothing is not allowed.


### 题解分析

这题就是判断给定的括号串是否合法的变形题，题目要求必须交换两个位置的字符，因此大的思路还是一样，采用栈保存左括号，然后遍历给定的字符串，分情况处理：

1. 栈为空：

	（1） s[i]为'('，一定是进栈。

	（2） s[i]为')'，一定要交换，显然肯定是跟'('交换，因此这里改为')'，然后进栈，标记为changed置为true，表示已经进行交换了，至于跟谁交换，不用关心。如果changed已经为true，说明之前交换过一次了，按照题意只能交换一次，因此无解直接break跳出循环。

2. 栈不为空：

	（1） s[i]为'('，一定是进栈。
	
	（2） s[i]为')'，跟栈顶的'('匹配，栈顶元素出栈。

遍历结束后，对结果进行讨论分析：

1. 遍历完整个字符串，changed为false，栈为空，也就是说输入给的字符串本来就说合法的括号匹配序列，不需要交换，但因为题目要求一定要进行一次交换，因此这里需要特殊处理一下：

	（1）字符串长度>2，显然我可以随便交换两个')'或者'('从而达到题目的要求，因此输出"Yes"。
	
	（2）字符串长度<=2，那没办法，输出"No"。

2. 遍历完整个字符串，changed为true，栈不为空而且元素个数为2，这时如果栈的元素都是'('，也就是2个'('，因为刚才我们把')'换成了'('，因此这里的第二个'('必须换成')'，也就是说栈的元素最终由两个'('变成一个'('和一个')'，刚好匹配，因此输出"Yes"。

3. 其它情况都输出"No"。

### AC代码

	#include <iostream>
	#include <string>
	#include <cstring>
	#include <iostream>
	#include <fstream>
	#include <algorithm>
	#include <map>
	#include <stack>
	
	using namespace std;
	
	int main() {
	
	    int t;
	    int n;
	    string s;
	    cin >> t;
	    while (t--) {
	        cin >> n;
	        cin >> s;
	        stack<char> st;
	        bool changed = false;
	        int i = 0;
	        for (; i < n; i++) {
	            if (st.empty()) {
	                // 栈为空
	                if (s[i] == '(') {
	                    st.push('(');
	                } else {
	                    if (changed) {
	                        break;
	                    }
	                    changed = true;
	                    st.push('(');
	                }
	            } else {
	                // 栈不为空
	                if (s[i] == '(') {
	                    st.push('(');
	                } else {
	                    st.pop();
	                }
	            }
	        }
	        if (i == n && !changed && st.empty()) {
	            if (n > 2) {
	                cout << "Yes" << endl;
	            } else {
	                cout << "No" << endl;
	            }
	        } else if (i == n && changed && st.size() == 2) {
	            char c1 = st.top();
	            st.pop();
	            char c2 = st.top();
	            if (c1 == '(' && c2 == '(') {
	                cout << "Yes" << endl;
	            } else {
	                cout << "No" << endl;
	            }
	        } else {
	            cout << "No" << endl;
	        }
	    }
	
	    return 0;
	}
