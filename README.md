# Cleaning_Data_with_SQL-NashvilleHousingDataSet-
Comprehensive breakdown of how we use SQL to perform Data clean up by altering and updating tables (ALTER TABLE UPDATE Statements), working on a Housing Dataset from Nashville Tennessee


SELECT * FROM Nashville_Housing2

--1) Standardize the date format (in SaleDate)
-- first, convert SaleDate column to date using CAST:

SELECT SaleDate, CAST(SaleDate AS date) FROM Nashville_Housing2
--a) UPDATE SaleDate column using the above CAST query (but it did not work as shown below):

UPDATE Nashville_Housing2
SET SaleDate = CAST(SaleDate AS date)

--b) so we ALTER TABLE by creating new column called SaleDateConverted, before UPDATE:

ALTER TABLE Nashville_Housing2
Add SaleDateConverted DATE
--c) UPDATE table by inserting the CAST query from "a" above into the newly created SaleDateConverted column as follows:

UPDATE Nashville_Housing2
SET SaleDateConverted = CAST(SaleDate AS date)

-- Test the new column to see if it works :)
SELECT SaleDateConverted FROM Nashville_Housing2

--2) Populate Property Address data
/* WHen we observed the entire table, we realized that some PropertyAddress had NULL values but also, there is a direct relationship between the 
PropertyAddress and the ParcelID columns in which a ParcelID number, always has the same PropertyAddress. With this information, we can fill in
 the NULL values in the PropertyAddress by comparing ParcelID values!
*/
--a) we will begin by SELF JOINING the table to fill in NULL values in PropertyAddress from information in ParcelID (in Sheet 1$, then in Nashville_Housing2 tables):

SELECT 
    a.ParcelID, a.PropertyAddress, b.UniqueID, b.PropertyAddress
FROM Nashville_Housing2 a 
JOIN Nashville_Housing2 b 
    ON a.ParcelID = b.ParcelID 
    AND a.UniqueID <> b.UniqueID -- (because the UniqueID is unique, and the JOIN will exclude multiple rows of ParcelID with different UniqueIDs)
WHERE a.PropertyAddress IS NULL

--b) We shall replace the NULL values in a.PropertyAddress with the values in b.PropertyAddress as follows:

SELECT 
    a.ParcelID, a.PropertyAddress, b.UniqueID, b.PropertyAddress, ISNULL(a.PropertyAddress, b.PropertyAddress)
FROM Nashville_Housing2 a 
JOIN Nashville_Housing2 b 
    ON a.ParcelID = b.ParcelID 
    AND a.UniqueID <> b.UniqueID 
WHERE a.PropertyAddress IS NULL
--c) Now let's UPDATE the table by replacing the NULLS with the derived values from the SELF JOIN as follows:

UPDATE a --(when UPDATING JOINS, you need to use the alias!)
SET PropertyAddress = ISNULL(a.PropertyAddress, b.PropertyAddress)
FROM Nashville_Housing2 a 
JOIN Nashville_Housing2 b 
    ON a.ParcelID = b.ParcelID 
    AND a.UniqueID <> b.UniqueID 
WHERE a.PropertyAddress IS NULL

--3) Splitting addesses into individual columns of (Address, City and State in Sheet1$ and Nashville_Housing2 tables respectively):

SELECT 
    SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1) as Address,
    SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress)) as City
FROM Nashville_Housing2

--Now ALTER TABLE and UPDATE table with the newly created split columns (of "Address" and "City") with the ALTER Aand UPDATE statements below:

ALTER TABLE Nashville_Housing2
Add PropertySplitAddress NVARCHAR(255)

UPDATE Nashville_Housing2
SET PropertySplitAddress = SUBSTRING(PropertyAddress, 1, CHARINDEX(',', PropertyAddress) -1)

ALTER TABLE Nashville_Housing2
Add PropertySplitCity NVARCHAR(255)

UPDATE Nashville_Housing2
SET PropertySplitCity = SUBSTRING(PropertyAddress, CHARINDEX(',', PropertyAddress) +1, LEN(PropertyAddress))
--b)lets split the 'OwnersAddress column (into Address, city and State) 
--like we did the previous using two methods as shown below:

SELECT 
    SUBSTRING(OwnerAddress, 1, CHARINDEX(',', OwnerAddress) -1) Addy,
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
--ALTER TABLE and UPDATE with three newly created columns (that show breakdown of address into addy, city and state):

ALTER TABLE Nashville_Housing2
Add OwnerSplitAddress NVARCHAR(255)

UPDATE Nashville_Housing2
SET OwnerSplitAddress = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3) 

ALTER TABLE Nashville_Housing2
Add OwnerSplitCity NVARCHAR(255)

UPDATE Nashville_Housing2
SET OwnerSplitCity = PARSENAME(REPLACE(OwnerAddress, ',', '.'), 2) 

ALTER TABLE Nashville_Housing2
Add OwnerSplitState NVARCHAR(255)

UPDATE Nashville_Housing2
SET OwnerSplitState =  PARSENAME(REPLACE(OwnerAddress, ',', '.'), 1) 

--4)Change Y and N to Yes and No in "SoldAsVacant" column of Nashville_Housing2 Table
--first let's COUNT how many "SoldAvVacant" rows are (Yes, No, Y and N):

SELECT DISTINCT SoldAsVacant, COUNT(SoldAsVacant) as Count
FROM Nashville_Housing2
GROUP BY SoldAsVacant
ORDER BY 2
--Now we use CASE Statetment to replace values
SELECT 
    SoldAsVacant,
CASE 
    When SoldAsVacant = 'Y' THEN 'Yes'
    When SoldAsVacant = 'N' THEN 'No'
ELSE SoldAsVacant
END --as NEW_SoldAsVaccant
FROM Nashville_Housing2
--UPDATE "Nashville_Housing2" table with the new CASE Statement as shown below:
UPDATE Nashville_Housing2
SET SoldAsVacant = CASE 
    When SoldAsVacant = 'Y' THEN 'Yes'
    When SoldAsVacant = 'N' THEN 'No'
ELSE SoldAsVacant
END

--5) Remove Duplicates
--The ROW_NUMBER is a ranking system based on duplicate rows found in the table.
--The PARTITION BY Clause included the following columns (ParcelID, ParcelID, PropertyAddress, SaleDate, SalePrice, LegalReference) to 
--compare if their values are all the same (hence duplicates), then they should be partitioned:

SELECT 
    *, 
    ROW_NUMBER() OVER(
    PARTITION BY   
        ParcelID,
        PropertyAddress,
        SaleDate,
        SalePrice,
        LegalReference
    ORDER BY UniqueID
    ) as Row_Num
FROM Nashville_Housing2
--Now we use a CTE (Common expresion table) as a temporary table that can be referenced or that new queries can be written from as shown below:
WITH RowNumCTE as(
SELECT 
    *, 
    ROW_NUMBER() OVER(
    PARTITION BY   
        ParcelID,
        PropertyAddress,
        SaleDate,
        SalePrice,
        LegalReference
    ORDER BY UniqueID
    ) as Row_Num
FROM Nashville_Housing2
)
SELECT *
From RowNumCTE
WHERE Row_Num > 1
ORDER BY PropertyAddress
--Now lets DELETE the "Row_Num" that are greater than 1 (Row_Num > 1) because they are the duplicate values. This is shown below:

WITH RowNumCTE as(
SELECT 
    *, 
    ROW_NUMBER() OVER(
    PARTITION BY   
        ParcelID,
        PropertyAddress,
        SaleDate,
        SalePrice,
        LegalReference
    ORDER BY UniqueID
    ) as Row_Num
FROM Nashville_Housing2
)
DELETE
From RowNumCTE
WHERE Row_Num > 1

--6) DELETE unused columns (but one has to be careful what columns you delete because of legal reasons at work):

ALTER TABLE Nashville_Housing2
DROP COLUMN OwnerAddress, TaxDistrict, PropertyAddress, SaleDate

-----------------------------------------------------------------------------------------------------------------
----------------------------------------------------THE END------------------------------------------------------
-----------------------------------------------------------------------------------------------------------------


