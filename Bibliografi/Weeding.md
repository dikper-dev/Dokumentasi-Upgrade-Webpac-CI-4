# Weeding
## Modul

NOTE PAGTINATION COUNT DAN LIST BEDA
```mysql
SELECT weeding.weedingDate, weeding.weedingDesc, library.libraryTitle, library_dataunit.libraryMainNumber, staff.staffName, library.libraryID FROM weeding 
JOIN library_dataunit ON library_dataunit.libraryMainNumber = weeding.libraryMainNumber 
JOIN library ON library.libraryID = library_dataunit.libraryID 
JOIN staff ON staff.staffID = weeding.weedingStaffID 
WHERE weeding.weedingStaffID > 0 AND weeding.weedingStaffDelete = 0 AND library_dataunit.ownerID = 1 AND weeding.ownerID = 1 
ORDER BY staff.staffIsActive DESC, staff.staffName;

SELECT COUNT(*) as totalRows 
FROM weeding 
JOIN library_dataunit ON library_dataunit.libraryMainNumber = weeding.libraryMainNumber 
WHERE weeding.weedingStaffID > 0 AND weeding.weedingStaffDelete = 0 AND library_dataunit.ownerID = 1 AND weeding.ownerID = 1;
```