[Move Zeroes](https://leetcode.com/problems/move-zeroes):  swap zero to the next non-zero to the right, zero nartually shift right while other non-zero will shift left
[Majority Element](https://leetcode.com/problems/majority-element) : frequency counting
[Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array): if n[i]== to_check_duplicate: replace n[i] by _ else: n[i] = to_check_duplicate
[Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self): prefix product (just like prefix sum) and postfix product, O(2n)
[Number of Zero-Filled Subarrays](https://leetcode.com/problems/number-of-zero-filled-subarrays) :  1+2+3+...+k=2k∗(k+1)​
[Increasing Triplet Subsequence](https://leetcode.com/problems/increasing-triplet-subsequence): - `first` = smallest number in `nums[0..i]`, `second` = smallest number in `nums[0..i]` such that `second > first`. Once we see a number `num > second`, we return true.
[Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii): every buy day ideally will have future higher-price sell day, note that if prices are 1, 2, 3, 4 then the most ideal buy-sell days are 1-4.
[Rotate Array](https://leetcode.com/problems/rotate-array): newIndex=(i+k)mod n; for O(1) space: reverse whole array -> reverse the first k elements -> reverse the remaining n-k elements.
[First Missing Positive](https://leetcode.com/problems/first-missing-positive): kho nhu cho deo hieu gi het
[Is Subsequence](https://leetcode.com/problems/is-subsequence): s always has length smaller than length of t -> easy
[Valid Palindrome](https://leetcode.com/problems/valid-palindrome):  use x.isalnum() to check if x is number. similar functions: isdigit(), isalpha(), isascii(), issuper(), islower()


