# MIM Spell Hack Review

The protocol did everything right. They rounded in the protocol's favour whenever they should but one additional function, meant to only reduce the user's funds, ended up enabling the attack. How?

<!-- display image -->

![image](https://pbs.twimg.com/media/GFJrzsnbkAAaBMy.png)

The protocol used shares mechanism to calculate the current debt of a user. Note: In the codebase, borrow shares are referred to as base/part and borrow assets amounts are referred to as elastic/amount.

When a user borrows certain funds, they get minted borrow shares based on the current totalBorrowAssets and totalBorrowShares ratio. As interest is owed from the user, totalBorrowAssets increases without totalBorrowShares increasing and in turn it increases the proportional amount that the user has to repay as well. The culprit function was the repayForAll function. This allowed anyone to repay everyone's debt. To accomplish this, the protocol reduced totalBorrowAssets without totalBorrowShares in repayForAll function.

The attacker borrowed funds using a flashloan and use the repayForAll function first.

---

<!-- display image -->

![image](https://pbs.twimg.com/media/GFJr0z3awAAAruT.jpg)

The attacker couldn't repay all of the amount as there was a check that totalBorrowAssets needed to be greater than 1000 ether. Regardless, the attacker repaid as much as they could, which reduced totalBorrowAssets such that totalBorrowAssets:totalBorrowShares was ~ 1:26.
Now, the attacker repaid all the existing loans for all the borrowers.

For the last remaining borrower, the user repaid all but 100 wei of shares.

At that point, totalBorrowedShares were 100, and totalBorrowAssets were 3.

<!-- display image -->

![image](https://pbs.twimg.com/media/GFJr1kaasAEn6PL.jpg)

The attacker repaid 1 wei of share 3 times, which meant totalAssets reduced to 0, and totalBorrowShares were 97. Now the attacker started borrowing 1 wei from their account. At this point, TotalBorrowAssets is 1, and totalBorrowShares is 98.
Attacker put up a very low amount of collateral (100 wei) and in a loop, started borrowing 1 wei of assets and repaying 1 wei of borrow shares.

<!-- display image -->

![image](https://pbs.twimg.com/media/GFJr2WAbMAA6JkN.jpg)

Since totalBorrowShares are greater than totalBorrowAssets, borrowing 1 wei of assets minted a lot of borrow shares.
However, since the protocol rounds up in its favour, when the attacker repaid 1 wei of borrow share, even though it is worth near 0, the protocol makes the attacker repay 1 wei of assets.
So at the end of this loop, totalBorrowAssets stays at 1, but totalBorrowShares increase exponentially.

The attacker made totalBorrowShares go up to infinity while keeping totalBorrowAssets at 1.
Now, they repaid 1 wei of share to make the totalBorrowAssets go to zero, and totalBorrowShares are still at infinity. Now, the attacker borrowed all of the funds in the protocol for collateral worth almost nothing using another account, and the protocol let it.
Why?

The attacker borrowed all of their funds, but the borrow shares that they got minted were at the ratio of 1:1.

<!-- display image -->

![image](https://pbs.twimg.com/media/GFJr3slaoAAY7s9.jpg)

Because in the borrow function when assets get converted in borrow shares, if totalBorrowAssets are zero, borrow shares minted are equal to the asset amount. So, the attacker borrowed a finite amount (all of the available borrowable amount), and they got finite borrow shares.

Remember, at this point, totalBorrowShares are still infinite. So when the protocol checks the health of this attacker account, they convert borrow shares into borrow amount to see how much they have borrowed.

This comes out to almost nothing as the shares that this attacker account has are nothing compared to shares that the other attacker account has. The protocol thinks that the other account, which has infinite borrow shares, has borrowed almost all the funds and not this account.

And the health check passes.

Fix: For lending protocols that use shares calculation to calculate owed interest, make sure there is no way to reduce totalBorrowShares less than totalBorrowAssets.
