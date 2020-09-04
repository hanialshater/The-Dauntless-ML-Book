# Pack the knapsack

We all like puzzels, I recall back when I was in school and fist time intreduced to 7 bridges of konigsberg, the famus problem from Lionardo Euler time where there are 7 bridges in a town and you need to take a walk through, visiting each part of the town and crossing each bridge only once. I have spend few hours with friends trying to understand how to do it. You will start discovring the brute force algorithm, then start recognizing patterns that make the brute force faster. We all like to be smart, solve probems and thats how I end up diving deep into progamming, mathmatics and algorithms. Now, 15 years later, almost all problemss are the same, you have a problem, brute force solition and you need to recognize patterns to solve it faster. 

Except the psychological appeal and pleasure of feeling smart beating the problem, I found the problem it self that matters not the solution. lets face it, we are living in exponential time, there is an explosion of number of businesses and problems to solve, just in the last decade we all saw the raise of social media, mobile apps, distributed systems, big data, AI, internet of things, electric cars, space technology and many other fields. I only have finite time to live, and if you are mortal like me, am sure you want to spend that budget of you life time to solve as much problems as you can instead of being smart in solving few problem in a fantasic smart way. To be hounest, it is not easy to let go all that passion in problem solving and accept that AI can do it better. I will give you a moment to console your self, however remember that we are here to tame the machine and not to admit defeat.


::: info
<p>Side info: Human vs. machines</p>

- the calclator
- deep blue
- alpha go
- Watson 
- NAS
- Algorihtm configuration
-batata 

:::

So how to side problem solving to the machine? The key skill you need to master is problem modeling, instead of solving the problem itself, you will recognise a problem pattern and state the problem in a suitable form that help machines to solve it. For simple cases like shortest path, you can directly use algorithm like dikstra to find the solution. However, in many real world scenarios, the problem will be too complex to solve by a standard algorithm or even NP-Complte, and in those cases you will use optimization or AI to do the job. Without further do lets take an example:

#### Knapsack problem
Suppose you are developing a trip plan and want to take some items with you in a knapsak, however, because you only can cary up to limited weight, you want to pick the items that are most valuable for you. suppose you have a hammer, wrench, screwdriver, towel. The weights and importance of these items are as follows.

| item        | Value | Weight |
|-------------|-------|--------|
| hammer      | 8     | 5      |    
| wrench      | 3     | 7      |
| screwdriver | 6     | 4      |
| towel       | 11    | 3      |

The obvious brute force algorithm is to try all combinations and find the one of best score, obviously the problem complexity is $O(2^n)$. For a variant of this problem where weight capacity is integer W, this problem can be solved with $O(N*W)$ complexity based on dynamic progamming the solution is as follows: for n items and w_max maximum weight we build a table of n rows and w_max columns to present solution value, each i, w entry in the table represent the max value till item i and weight w. the initial value with zero items is zero for all weights, for item i and weight j the solution value `sln[i, j] = max(val[i] + sln[i-1][w - item_w[i]], sln[i-1][w])`



```{.python .input  n=72}
import numpy as np
sln = np.zeros([n, w])

def knapsack(w_max, item_w, item_val, n): 
    sln = np.zeros([n + 1, w_max + 1])
    for i in range(1, n + 1): # first row no items, need n + 1 rows 
        for w in range(1, w_max + 1): # first column is 0 weight need w_max + 1 columns
            if item_w[i] < w:
                sln[i][w] = max(val[i] + sln[i-1][w - item_w[i]], sln[i-1][w])
            else: 
                sln[i][w] = sln[i - 1][w] 
    print(sln)
    return sln[n][w_max] 
  
# # Driver program to test above function 
wt = [0,5, 7, 4, 3]
val = [0, 8, 3, 6, 11]
n = len(val) - 1
w_max = 14

print(knapsack(w_max, wt, val, n)) 
```

```{.json .output n=72}
[
 {
  "name": "stdout",
  "output_type": "stream",
  "text": "[[ 0.  0.  0.  0.  0.  0.  0.  0.  0.  0.  0.  0.  0.  0.  0.]\n [ 0.  0.  0.  0.  0.  0.  8.  8.  8.  8.  8.  8.  8.  8.  8.]\n [ 0.  0.  0.  0.  0.  0.  8.  8.  8.  8.  8.  8.  8. 11. 11.]\n [ 0.  0.  0.  0.  0.  6.  8.  8.  8.  8. 14. 14. 14. 14. 14.]\n [ 0.  0.  0.  0. 11. 11. 11. 11. 17. 19. 19. 19. 19. 25. 25.]]\n25.0\n"
 }
]
```

```{.python .input}

::: info
<p>Side info: NP completeness and complexity calasess</p>

:::

If you noticed, comming with solution for knapsak problem and computing is not intutive and even for trained person it will require few time of thinking. In real world problems, slight variants can make the problem even harder, for example suppose you are desining a cretical trip (like trip to boton of the ocain) and you need to get an idea about components to consider for with max weight 300Kgs, you will have extra constraints like you need to select specific compnents. (TODO refine)

Now lets solve the problem throgh optmization, we will use pyomo which is a great python package that wraps multiple optimizers and provide an elegant way of modeling optimization problems.

First thing we define the data model, it captures the item, the importance, and the weight of each item.
```

```python
# the data
A = ['hammer', 'wrench', 'screwdriver', 'towel']
b = {'hammer':8, 'wrench':3, 'screwdriver':6, 'towel':11}
w = {'hammer':5, 'wrench':7, 'screwdriver':4, 'towel':3}

```


To model the problem we should define a few other constructs:
**Optimization variable**: in this case, a boolean variable per items that indicate if it should be selected for the trip or not.

```python
model.x = Var( A, within=Binary )
```


**Objective function**: in this case, the sum of values of items that we can carry

```python
model.value = Objective(expr = sum( b[i]*model.x[i] for i in A), sense = maximize )
```


**Constraints**: that are imposed on our problem, here the sum of picked items wait are less than 14

```python
W_max = 14
model.weight = Constraint(expr = sum( w[i]*model.x[i] for i in A) <= W_max )
```


Fortunately, these constructs are available in pyomo and are intuitive to use. You can just define variables, objectives, and constraints and attach them to a model. After that, you just initiate a solver and voila, you have your answer!

```{.python .input  n=2}
from pyomo.environ import *
import pyomo.environ as pe 

# the data
A = ['hammer', 'wrench', 'screwdriver', 'towel']
b = {'hammer':8, 'wrench':3, 'screwdriver':6, 'towel':11}
w = {'hammer':5, 'wrench':7, 'screwdriver':4, 'towel':3}
W_max = 14

#the model 
model = ConcreteModel()

# the variables
model.x = Var( A, within=Binary )

# the objective 
model.value = Objective(expr = sum( b[i]*model.x[i] for i in A), sense = maximize )

# weights 
model.weight = Constraint(expr = sum( w[i]*model.x[i] for i in A) <= W_max )

#solver 
opt = SolverFactory('glpk')

#Voela !
result_obj = opt.solve(model) 
model.x.get_values()

```

```{.json .output n=2}
[
 {
  "data": {
   "text/plain": "{'towel': 1.0, 'screwdriver': 1.0, 'wrench': 0.0, 'hammer': 1.0}"
  },
  "execution_count": 2,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```

#### Modifying the problem
now let's say you decided that either hammer or wrench should be picked. No problems, manipulate the model, define a new constraint, run again and that's it. You can see now that instead of focusing on the algorithm itself, you have room to focus on the problem and let the optmizer do the heavy lifting.

```{.python .input  n=3}
model.c = Constraint(expr = sum(model.x[i] for i in [A[1], A[3]]) == 1)
result_obj = opt.solve(model) 
model.x.get_values()
```

```{.json .output n=3}
[
 {
  "data": {
   "text/plain": "{'towel': 1.0, 'screwdriver': 1.0, 'wrench': 0.0, 'hammer': 1.0}"
  },
  "execution_count": 3,
  "metadata": {},
  "output_type": "execute_result"
 }
]
```
