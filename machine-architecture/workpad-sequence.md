# Workpad Sequence

A Workpad, in its simplest form, is the story of a single exchange, recorded in binary using standardized sets of 32 and 64 bits. Its standard sequencing ensures reliable and resilient transmission using all mediums such as bluetooth, radio and serial, free from any dependencies on supplementary data. The record interprets the record.

The compression of exchange information into binary reduces the use of electricity at all scales, allowing for better performance of all devices involved in the creation, storage and transmission of business records. It also achieves greater fidelity over large distances.

Each exchange has a value as well as a quantity which is oftentimes simply "1". Treatment of the value and quantity will be reviewed in the next section.

For both participants in the exchange, the value must be accounted for using general principles. Does the value represent an increase or decrease in assets, expenses, or sales? Is the exchange equity-related? Does the exchange value, in whole or in part, represent a liability? The variable types of exchange are: Sale Complete, Sale Pending, Expense Complete, Expense Pending, Equity Increase Complete, Equity Increase Pending, Equity Decrease Complete, and Equity Decrease Pending. Sales and Expenses are often refunded which must also provide for. Regular Sales should be differentiated from Other Sales at minimum. Expenses for Sales (ie Cost of Goods Sold) and Operational Expenses also benefit from special markers. This together provides for a basic chart of accounts which we can number this way:

1. Assets
2. Sales
3. Other Sales
4. Sales Expenses
5. Operating Expenses
6. Equity (ie Investment)
7. General Liabilities
8. Authority Liabilities

Most of these variations can be provided for using a simple 3 flag scheme. For a short Accounting Sequence of bits, let us assume:

* the first position represents 0 as Out and 1 as In
* the second position represents 0 as Past and 1 as Future
* the third position represents 0 as Out and 1 as In
* the first position symbolizes the business or entity
* the third position symbolizes the owner or operator when position 1 and 3 are opposite

Using this scheme above:

* In Past In represents Sale with Assets Increased
* In Future In represents Future Sale with Assets Increased
* In Past Out represents Equity Increase with Assets Increased
* In Future Out represents Future Equity Increase with Assets Increased
* Out Past In represents Expense with Equity Decreased
* Out Future In represents Future Expense with Equity Decreased
* Out Past Out represents an Expense with Assets Decreased
* Out Future Out represents Future Expense with Assets Decreased

This gives us a notation style of "I > O", "O < O", etc.

We must still allow for identifying types of sales (sales, other), expenses (COGS, operating) and liabilities (expense, equity).

All future-coded exchanges represent 1 of 4 liability types:

* Sale
* Equity Increase
* Equity Decrease
* Expense

In this simplified accounting system, it suffices to assume that all liabilities pertain to the future regardless of their initially intended date of settlement.

A mechanism for identifying if a liability has matured and by how much will be identified.

Each record may belong to a set of similar records in a larger collection or transmission, so for a minimal approach we can provide for 3 levels of association per the ASCII control characters:

1. File (28,FS)
2. Group (29,GS)
3. Record (30,RS)

Inclusion of Unit Separator (31,US) will be demonstrated to be unnecessary for a robust system.

The last principle needee for an exchange record is the representation of the entity transmitting, and possibly their sub-identity which may indicate they are serving as a representative.

With these points established we can return to the numerical elements of the record and the various flags which serve as modifiers for the different kinds of value and quantity combinations. Follewing that we can deal with unique identifiers and the larger sequences of bits that compose a Workpad.

NUMERICAL POSSIBILITIES & SEQUENCE NAMES

The 32 bit sequence that carries the primary data about the exchange is called Record Value (RV) and it's followed by a dependent sequence named Record Treatment (RT).

To maximize the performance capacity and efficiency of RV and RT, we begin by dividing RV into two parts, a 25 bit portion dedicated to value and quantity, and the remaining 7 bits to critical flags.

The 7 critical flags of RV are: 26th bit: Scaling Factor? 27th bit: Optimal Split? 28th bit: Order of Split? 29th bit: In or Out for the Entity? 30th bit: Past or Future Exchange? 31st bit: Equity Related? 32nd bit: Undetermined

These are followed by additional flags and numeric blocks in RT as follows:

1 - 33rd to 34th bits: Sender Format Requiring Conversion, Converted, Copy from Sender, For Represented Entity 2 - 35th to 41st bits: Scaling Factor of maximum 127 3 - 42nd to 45th bits: Optimal Split of maximum 15 indicates size of multiplier within RV 25 bit block 4 - 46th and 47th bits: Enquiry and Acknowledge Bells 5 - 48th to 59th bits: Files, Records, Groups Separators with a variable ratio of bits, defaulting to 3:5:4 6 - 60th to 64th bits: Sub-Identities of maximum 31, enlargeable by ratio of Separators

This pattern of the initial 64 bits is assumed by default, unless a Setup Sequence initiates the new transmission using a patterned alternation of 1s and 0s, appended below.

Scaling Factor can be referred to as Quantity1 and Optimal Split amount can be termed Quantity2. The base value of the 25 bit block is our Multiplicand and the 2nd portion of this block is the Multiplier, otherwise defined as M1 and M2. Let the 1st flag at 26th bit represent Off or On for Quantity1 (Block 2 - Scaling Factor) and 27th bit represent Quantity2 (Block 3 - Optimal Split Amount). So if Q1 set to ON then assumption is: 25 bits carry a max Base Value, 33,554,431 and 3rd sub-block in RT represents a Quantity. But for majority of economic transactions max 25 bit value is greater than what's required, i.e. a division of the 25 bits suffices to create representation for the Value and Quantity -- with equilibrium determined by a dedicated algorithm and Optimal Split block utilized for specificity.

```
Continuing with flag modification, the setting of Bit 27 to ON would confirm inclusion of 4th 	sub-block in RT with alias Q2. Does this confirm activation of "Max Formula" ie. M1xM2xQ1xQ2? The 	7th possible flag at Bit 32 in RV is the last option for creating a varied calculation. This flag 	could possibly indicate that 1 of the 4 numbers represents a fixed quantity and therefore remains 	a fixed number in the Max Formula.

FLAG: How do we know if maxed 25bit value with using of Optimal Split or Scaling Factor 		represents value * quantity or the Very High Value of quantity q?
```

What does the 25 bit plus 7 plus 4 setup achieve, aside from its performance as an equation that can calculate all integers up to 4.54 billion? How we can achieve these calculations without any gaps of integers will be appended. Our approach of 4 sets of numbers allows any integer up to 33,554,431 to be provided for with just the use of 25 bits, allowing for Flags 26 and 27 to be kept off. Where units are 1023 or less and cost is 32,767 or less, only the 25 bits are required to make the record. When value is e.g. in the range of 4194303, quantity can be max 7 while staying within 25 bits and thereby not depending on Record Treatment block. This achieves a high degree of resiliency in regards to the majority of transmissions and their potential interruption.

```
If we shift our approach for some scenarios demarcated by flags -- and let the 25 bits represent 	a max common value then the Scaling Factor could be treated as the quantity which is max 127, 	which would cover most purchasing scenarios in most economies.
```

Program must perform an ordered set of checks based on the submitted numbers. IF below 33.554M, is value and quantity expressible from an ideal division of 25 bits? IF value is less than 33.554 and the factor of its quantity becomes greater than 33,554 then the use of Q1 and maybe Q2 are required to encode an equation representing the total. IF Value requires 19-25 bits and quantity is 127 or less, then Alt Cal 1 i.e. AC1 selected with Q1 flag set at bit 26.

The order of numerical possibilities is as follows:

IF value is 16,777,215 or below and quantity is 1 THEN SF set to Off and OS set to On OS block (4 bits) combines with Scaling Factor, indicates that M2 is value 1 Series of bits can be zeroed out or "00000000001"

IF value is between 8,488,608 and 16,777,215 and quantity is 2 to 2047 THEN SF set to Off and OS set to ON OS combines with SF block, indicates M2 is in range of 2-2047 E nb IF value is 33,554,431 or below and quantity is 1-2047 TeEN flags 26 and 27 for SF and OS each set to Off ("0") If Quantity = 1, series of bits can be zeroed out or "00000000001" Otherwise, quantities 2-2047 represented using the unified set of 11 bits

IF Scaling Factor (SF) bit set to On and Optimal Split (OS) bit set to Off THEN we assume the Multiplicand (M1) factored by SF with OS representing Quantity SO

IF SF bit set to On and Optimal Split set to On THEN we assume M1 factored by SF and OS represents division of the 25 bits, wherein the remainder of that bit set (M2) represents Quantity

IF SF bit set to Off and OS set to Off THEN we assume M1 taken up to maximum with SF and OS combined representing Quantity

```
This flag could indicate that 1 of the 4 numbers represents a fixed quantity and therefore 	remains a fixed number in the Max Formula.
```

Treatment of Numbers from 33,554,432 to Highest Consecutive Integer

Formula for Determining Reverse Engineering Highest Consecutive Integer

Treatment of Dollars & Cents ie Exchange Values with Fractional Integers Confirm method for determining 3 decimal place accuracy up to 16 million and past
