---
title: '排列与组合'
date: 2020-09-07 14:37:00
tags: [算法,排列与组合]
published: true
hideInList: false
feature: 
isTop: false
---

关于排列与组合的算法问题

## 排列与组合的区别

实际上，排列意味着数据一样，数据出现的顺序不一样；
组合意味着，数据不一样，与数据的顺序无关；

## 排列的算法框架

如果数据没有重复，直接套用回溯算法框架即可；

```C++

class Solution {
public:

    void Help(vector<vector<int>>& results, vector<int>& result, vector<int>& num, vector<int>& flags)
    {
        if (result.size() == num.size())
        {
            results.push_back(result);
            return;
        }

        for (int i = 0; i < flags.size(); i++)
        {
            if (flags[i] == 0)  //跳过已经取过的数
                continue;

            result.push_back(num[i]);
            flags[i] = 0;
            Help(results, result, num, flags);
            flags[i] = 1;
            result.pop_back();
        }
    }

    vector<vector<int>> permute(vector<int>& nums) {
        vector<vector<int>> results;
        vector<int> result;
        vector<int> flags(nums.size(), 1);;
        Help(results, result, nums, flags);

        return results;
    }
};
```

对于特殊情况，比如数据中有重复的数据，需要考虑重复的情况；为了方便识别重复的情况，需要对数据源进行排序，这样在计算过程中，对于重复的情况，只需要向前单向查找是否重复即可；

## 组合的算法框架

如果直接套用回溯框架，就会导致重复问题，因为组合与顺序无关，先选跟后选可能会产生同一组数据；

解决办法：在回溯的过程中，只向后查找，而不向当前节点的前面查找，这样就不会产生重复的组合；

```c++
class Solution {
public:

    void FindAll(vector<int>& candidates, vector<vector<int>>& results, vector<int> &result, int &current, int &target, int id)
    {
        if (current == target)
        {
            results.push_back(result);
            return;
        }

        for (int i = id; i < candidates.size(); i++)
        {
            if (candidates[i]  <= target - current )
            {
                result.push_back(candidates[i]);
                current += candidates[i];
                FindAll(candidates, results, result, current, target, i);
                current -= candidates[i];
                result.pop_back();
            }
            else
            {
                break;
            }
        }
    }

    vector<vector<int>> combinationSum(vector<int>& candidates, int target) {

        vector<vector<int>> results;
        vector<int> result;
        int current = 0;

        FindAll(candidates, results, result, current, target, 0);

        return results;
    }
};
```

同样对于数据中有重复的数据，需要考虑重复的情况；需要对数据源进行排序，在计算过程中，对于重复的情况，只需要向前单向查找是否重复即可；
