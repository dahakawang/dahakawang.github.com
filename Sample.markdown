---
layout: post
title: "sample code"
date: 2013-03-11 22:58
comments: true
sharing: false
categories: 
published: false
---

This is the title
===========================
hello ``` ruby x = x + 1 ```this is what i wannt to say hello this is what i wannt to say hello this is what i wannt to say  
pin hello this is what i wannt to say hello this is what i wannt to say

{% codeblock test for ruby code lang:ruby https://github.com/dahakawang/for-wangjohn-to-see GitHub%}
def fact(n)
  if n == 0
    1
  else
    n * fact(n-1)
  end
end
{% endcodeblock %}

``` ruby test for ruby code https://github.com/dahakawang/for-wangjohn-to-see GitHub
def fact(n)
  if n == 0
    1
  else
    n * fact(n-1)
  end
end
```
> A sample blockquote.
>
> >Nested blockquotes are
> >also possible.
>
> ## Headers work too
> This is the outer quote again.


| A simple | table |
| with multiple | lines|


| Header1 | Header2 | Header3 |
|:--------|:-------:|--------:|
| cell1   | cell2   | cell3   |
| cell4   | cell5   | cell6   |
|----
| cell1   | cell2   | cell3   |
| cell4   | cell5   | cell6   |
|=====
| Foot1   | Foot2   | Foot3
{: rules="groups"}


$$
\begin{align*}
  & \phi(x,y) = \phi \left(\sum_{i=1}^n x_ie_i, \sum_{j=1}^n y_je_j \right)
  = \sum_{i=1}^n \sum_{j=1}^n x_i y_j \phi(e_i, e_j) = \\
  & (x_1, \ldots, x_n) \left( \begin{array}{ccc}
      \phi(e_1, e_1) & \cdots & \phi(e_1, e_n) \\
      \vdots & \ddots & \vdots \\
      \phi(e_n, e_1) & \cdots & \phi(e_n, e_n)
    \end{array} \right)
  \left( \begin{array}{c}
      y_1 \\
      \vdots \\
      y_n
    \end{array} \right)
\end{align*}
$$


![my icon](http://i.minus.com/i01yeD3zCWPoV.jpg)

![my icon](/images/icon.jpeg)