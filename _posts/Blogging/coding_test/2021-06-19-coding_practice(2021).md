---
title: Running Sum of 1d Array(leet code)
author: JaeHa
date: 2021-06-19 15:00:00 +0800
categories: [Blogging, coding_test]
tags: [coding_test]
pin: true
---
<br/>
<br/>
### 1480 Running Sum of 1d Array
<br/>
<br/>

```c++

class Solution {
public:
    vector<int> runningSum(vector<int>& nums) {
        
        int result = 0 ;

        vector<int> result_final;
                
        for( int i = 0; i < nums.size(); i ++)
        
        {
            result += nums.at(i);
            
            result_final.push_back(result);
            
        }
            return result_final;
        }

};

```