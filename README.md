# Project_Color_Paint_Company
XYZ Paints Inc. is a paint company which was recently launched in 2019. You have to work on some of the pain points of the marketing team like consumer growth, lead conversions, etc. as well as help them work on new set of campaigns or doing market basket analysis.

---To find out the maximum and minimum transactions on a year basis---
select a.*, max(total_Transaction) over () as max_, 
            min(total_Transaction) over () as min_
from (
      select count(*) as total_Transaction ,extract(year from PurchaseDate) as Purchase_year
from CustomerTransactionData 
group by Purchase_year ) as a;


---To find out the maximum and minimum transactions on a Quarter basis---
select a.*, max(total_Transaction) over () as max_, 
            min(total_Transaction) over () as min_
from (
      select count(*) as total_Transaction ,extract(Quarter from PurchaseDate) as Purchase_Quarter
from CustomerTransactionData 
group by Purchase_Quarter ) as a;


---To find out the maximum and minimum transactions on a Monthly basis---
select a.*, max(total_Transaction) over () as max_, 
            min(total_Transaction) over () as min_
from (
      select count(*) as total_Transaction ,extract(Month from PurchaseDate) as Purchase_Quarter
from CustomerTransactionData 
group by Purchase_Month ) as a;



------Identify the total_purchase order by Item/Product category-----
select c.Amt, Item_Category from Item as i 
        join 
        (
        select item_id, sum(PurchasingAmt) as Amt
          from CustomerTransactionData
          group by item_id
          order by Amt desc) as c
          on c.item_id = i.Item_Id;


------Identify the total_purchase on the basis of OrderType-----
select sum(PurchasingAmt) as amt, OrderType 
from CustomerTransactionData
group by OrderType
order by amt desc;


----To get the total purchasing amount on the basis of City_Id and City_Tier-----
select e.*, cd.CityTier from CityData as cd 
          join 
(select t.amt, c.City_Id from Customer as c 
          join 
    (select sum(PurchasingAmt) as amt , Cust_Id from CustomerTransactionData 
            group by Cust_Id) as t 
            on c.Customer_Id = t.Cust_Id ) as e
    on e.City_Id = cd.City_Id
    order by e.amt desc;


----Identify the total number of transactions with campaign coupon vs total number of transactions without campaign coupon. ----
select 
sum(
    case 
    when campaign_id is not null then 1 
    else 0
    end) as with_campaign,
            sum(
                 case
                 when campaign_id is null then 1
                 else 0
                 end) as without_campaign
                    from CustomerTransactionData;



---identify the dates when the same customer has purchased some product from the company outlets----
select P1.PurchaseDate as PurchaseDate1, p2.PurchaseDate as PurchaseDate2, P1.Cust_Id, P1.item_id as item1, p2.item_id as item2
from
  CustomerTransactionData as P1 
        join 
  CustomerTransactionData as p2 
      on P1.Cust_Id = p2.Cust_Id 
      where P1.Trans_Id <>p2.Trans_Id and P1.item_id <> p2.item_id and P1.OrderType = p2.OrderType
      order by P1.Cust_Id desc;


----Out of the first query where you have captured a repeated set of customers, please identify the same combination of products coming at least thrice sorted in descending order of their appearance.
select f.item1, f.item2, count(*) as cont 
from (
    select P1.PurchaseDate as PurchaseDate1, p2.PurchaseDate as PurchaseDate2, P1.Cust_Id, P1.item_id as item1, p2.item_id as item2
    from
        CustomerTransactionData as P1 
            join 
        CustomerTransactionData as p2 
              on P1.Cust_Id = p2.Cust_Id 
              where P1.Trans_Id <>p2.Trans_Id and P1.item_id <> p2.item_id and P1.OrderType = p2.OrderType
              order by P1.Cust_Id desc) as f 
              group by f.item1, f.item2;

------ Get the total discount, if any----
DELIMITER $$
CREATE FUNCTION Discount
(Quantity int, Price float, PurchasingAmt float)
RETURNS INT 
DETERMINISTIC
BEGIN
    DECLARE discount INT;
    SET discount = Quantity * Price - PurchasingAmt;
    RETURN discount; 
END$$
DELIMITER ;


----Get the days/month/year elapsed since the last purchase of a customer depending on input from the user.-----
DELIMITER $$
CREATE FUNCTION Time_Elapsed
(val varchar(4), date_last_purchase date)
RETURNS INT 
DETERMINISTIC
BEGIN
    DECLARE time_elapsed INT;
    SET time_elapsed = IF(val='day', DATEDIFF(NOW(), date_last_purchase), YEAR(NOW()) - YEAR(date_last_purchase));
    RETURN time_elapsed; 
END$$
DELIMITER ;



-----Identify whether a particular transaction amount (purchase amount) is 'correct' or 'not correct'.--

--It is correct if price and quantity are used to calculate without a coupon. In case of a coupon, the coupon amount should be deducted from the original amount given the ----original amount is greater than equal to min purchase for a coupon; else you can simply calculate original amount based on quantity. [Input will be transaction id]--
--[Note: Look out for null coupon ids]---

DELIMITER $$
CREATE PROCEDURE PurchaseAmountValidation (IN p1 varchar(32), OUT p2 varchar(128))
BEGIN 
    SELECT
    IF(PurchasingAmt != totalamt, 'not correct', 'correct') AS message
    INTO p2
    FROM (
	SELECT CT.PurchasingAmt,
    IF( CT.coupon_id IS NOT NULL AND Quantity * Price >= Min_Purchase,  Quantity * Price - IF(couponType != 'Flat',  Quantity * Price * Value * 0.01, Value), Quantity * Price) AS totalamt
    FROM
    Item AS I
    JOIN
    CustomerTransactionData AS CT
    ON I.Item_Id = CT.item_id
    LEFT JOIN CouponMapping AS CM
    ON CT.coupon_id = CM.coupon_id
    WHERE CT.Trans_Id = p1) AS T;
END $$
DELIMITER ;

CALL PurchaseAmountValidation('TID00240', @p2);
SELECT @p2;






