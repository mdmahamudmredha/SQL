```
SELECT * FROM students;

SELECT name AS Student_Name, class FROM students;

SELECT DISTINCT class AS DIS_Class FROM students;

SELECT DISTINCT name AS DIS_name FROM students;

SELECT DISTINCT gender FROM students;


-- AG Function
SELECT COUNT(*) AS TotalStudents FROM students;

SELECT COUNT(class) AS TotalStudents FROM students;

SELECT SUM(marks) AS TotalMarks FROM students;

SELECT ROUND(AVG(marks),1) AS AverageMarks FROM students;


SELECT MIN(marks) AS LowestMark FROM students;

SELECT MAX(marks) AS HighestMark FROM students;


SELECT COUNT(DISTINCT class) FROM students;
```
