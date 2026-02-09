[Move Zeroes](https://leetcode.com/problems/move-zeroes):  swap zero to the next non-zero to the right, zero nartually shift right while other non-zero will shift left
[Majority Element](https://leetcode.com/problems/majority-element) : frequency counting
[Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self): prefix product (just like prefix sum) and postfix product, O(2n)
def productExceptSelf(nums):
    n = len(nums)
    answer = [1] * n

    # Step 1: prefix products
    prefix = 1
    for i in range(n):
        answer[i] = prefix
        prefix *= nums[i]

    # Step 2: suffix products
    suffix = 1
    for i in range(n - 1, -1, -1):
        answer[i] *= suffix
        suffix *= nums[i]

    return answer
[Number of Zero-Filled Subarrays](https://leetcode.com/problems/number-of-zero-filled-subarrays) :
[Increasing Triplet Subsequence](https://leetcode.com/problems/increasing-triplet-subsequence):
[Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii): every buy day ideally will have future higher-price sell day, note that if prices are 1, 2, 3, 4 then the most ideal buy-sell days are 1-4.
[Rotate Array](https://leetcode.com/problems/rotate-array) 
[First Missing Positive](https://leetcode.com/problems/first-missing-positive): kho nhu cho deo hieu gi het
[Is Subsequence](https://leetcode.com/problems/is-subsequence): s always has length smaller than length of t -> easy
[Valid Palindrome](https://leetcode.com/problems/valid-palindrome):  use x.isalnum() to check if x is number. similar functions: isdigit(), isalpha(), isascii(), issuper(), islower()

