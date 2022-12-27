# Cleaning_Data_with_SQL-NashvilleHousingDataSet-
Comprehensive breakdown of how we use SQL to perform Data clean up by altering and updating tables (ALTER TABLE UPDATE Statements), working on a Housing Dataset from Nashville Tennessee


SELECT *
FROM Sheet1$ 

-- #1 STANDARDZE DATE FORMAT (IN SaleDate column)
-- first, convert SaleDate column to date using CAST
SELECT SaleDate, CAST(SaleDate AS date) 
FROM Sheet1$

--a) UPDATE SaleDate column using the above CAST query (but it did not work)
UPDATE Sheet1$
SET SaleDate = CAST(SaleDate AS date) 

--b) ALTER TABLE by creating new column called SaleDateConverted
ALTER TABLE Sheet1$
Add SaleDateConverted DATE
--c) UPDATE table by inserting the CAST query from "a" above into the newly created SaleDateConverted column as follows:
UPDATE Sheet1$
SET SaleDateConverted = CAST(SaleDate AS date) 

-- Test the new column to see if it works :)
SELECT SaleDateConverted
FROM Sheet1$

-- #2 Populate Property Address data
/* WHen we observed the entire table, we realized that some PropertyAddress had NULL values but also, there is a direct relationship between the 
PropertyAddress and the ParcelID columns in which a ParcelID number, always has the same PropertyAddress. With this information, we can fill in
 the NULL values in the PropertyAddress by comparing ParcelID values!
*/
--a) we will begin by SELF JOINING the table to fill in NULL values in PropertyAddress from information in ParcelID
SELECT 
    a.ParcelID, a.PropertyAddress, b.UniqueID, b.PropertyAddress
FROM Sheet1$ a 
JOIN Sheet1$ b 
    ON a.ParcelID = b.ParcelID 
    AND a.UniqueID <> b.UniqueID -- (because the UniqueID is unique, and the JOIN will exclude multiple rows of ParcelID with different UniqueIDs)
WHERE a.PropertyAddress IS NULL

--b) We shall replace the NULL values in a.PropertyAddress with the values in b.PropertyAddress as follows:
SELECT 
    a.ParcelID, a.PropertyAddress, b.UniqueID, b.PropertyAddress, ISNULL(a.PropertyAddress, b.PropertyAddress)
FROM Sheet1$ a 
JOIN Sheet1$ b 
    ON a.ParcelID = b.ParcelID 
    AND a.UniqueID <> b.UniqueID 
WHERE a.PropertyAddress IS NULL

--c) Now let's UPDATE the table by replacing the NULLS with the derived values from the SELF JOIN as follows:
UPDATE a -- (when UPDATING JOINS, you need to use the alias!)
SET PropertyAddress = ISNULL(a.PropertyAddress, b.PropertyAddress)
FROM Sheet1$ a 
JOIN Sheet1$ b 
    ON a.ParcelID = b.ParcelID 
    AND a.UniqueID <> b.UniqueID 
WHERE a.PropertyAddress IS NULL

-- #3 Splitting addesses into individual columns of (Address, City and State)
SELECT 
    SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1) as Address,
    SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress)) as City
FROM Sheet1$
--Now ALTER TABLE and UPDATE table with the newly created split columns (of "Address" and "City") with the ALTER Aand UPDATE statements below:
ALTER TABLE Sheet1$
Add PropertySplitAddress NVARCHAR(255)

UPDATE Sheet1$
SET PropertySplitAddress = SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1)

ALTER TABLE Sheet1$
Add PropertySplitCity NVARCHAR(255)

UPDATE Sheet1$
SET PropertySplitCity = SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress)) 
--On the second table (Nashville_Housing2), lets split the 'OwnersAddress column (into Address, city and State) 
--like we did the previous using two methods as shown below:
SELECT 
    SUBSTRING(OwnerAddress, 1, CHARINDEX(',', OwnerAddress) -1) ADDY,
    SUBSTRING(OwnerAddress, CHARINDEX('G', OwnerAddress), 14) CITY,
    SUBSTRING(OwnerAddress, CHARINDEX('TN', OwnerAddress), 3) STATE
FROM Nashville_Housing2
----------------or method 2 using PARSENAME------------------
--PARSENAME recognizes periods ('.') and not commas (','), so we have to replace the periods with commas first as hown below:
SELECT
    PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3) as Addy,
    PARSENAME(REPLACE(OwnerAddress, ',', '.'), 2) as City,
    PARSENAME(REPLACE(OwnerAddress, ',', '.'), 1) as State
FROM Nashville_Housing2

