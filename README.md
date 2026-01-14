# Advance_SQL
# advance_sql
Employee Performance &amp; Payroll Management System
/* ================================
   DATABASE CREATION
================================ */
CREATE DATABASE IF NOT EXISTS enterprise_db;
USE enterprise_db;

/* ================================
   TABLE CREATION
================================ */

CREATE TABLE departments (
    dept_id INT PRIMARY KEY,
    dept_name VARCHAR(50),
    location VARCHAR(50)
);

CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    dept_id INT,
    hire_date DATE,
    base_salary DECIMAL(10,2),
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);

CREATE TABLE attendance (
    emp_id INT,
    att_date DATE,
    status ENUM('Present','Absent','Leave'),
    PRIMARY KEY (emp_id, att_date),
    FOREIGN KEY (emp_id) REFERENCES employees(emp_id)
);

CREATE TABLE performance (
    emp_id INT,
    review_year INT,
    rating INT CHECK (rating BETWEEN 1 AND 5),
    PRIMARY KEY (emp_id, review_year),
    FOREIGN KEY (emp_id) REFERENCES employees(emp_id)
);

CREATE TABLE payroll (
    emp_id INT,
    pay_month DATE,
    bonus DECIMAL(10,2),
    deductions DECIMAL(10,2),
    net_salary DECIMAL(10,2),
    FOREIGN KEY (emp_id) REFERENCES employees(emp_id)
);

/* ================================
   SAMPLE DATA INSERTION
================================ */

INSERT INTO departments VALUES
(1,'IT','Delhi'),
(2,'HR','Mumbai'),
(3,'Finance','Bangalore');

INSERT INTO employees VALUES
(101,'Amit Sharma',1,'2020-02-10',60000),
(102,'Neha Verma',1,'2019-06-15',80000),
(103,'Rahul Singh',2,'2018-01-20',50000),
(104,'Pooja Mehta',3,'2017-09-05',90000),
(105,'Ankit Jain',3,'2021-03-25',70000);

INSERT INTO attendance VALUES
(101,'2025-01-01','Present'),
(101,'2025-01-02','Present'),
(102,'2025-01-01','Absent'),
(103,'2025-01-01','Present'),
(104,'2025-01-01','Leave'),
(105,'2025-01-01','Present');

INSERT INTO performance VALUES
(101,2025,5),
(102,2025,4),
(103,2025,3),
(104,2025,5),
(105,2025,4);

/* ================================
   ADVANCED SQL QUERIES
================================ */

-- 1. Top 3 highest paid employees per department
SELECT *
FROM (
    SELECT e.emp_name, d.dept_name, e.base_salary,
           RANK() OVER (PARTITION BY d.dept_id ORDER BY e.base_salary DESC) rnk
    FROM employees e
    JOIN departments d ON e.dept_id = d.dept_id
) x
WHERE rnk <= 3;

-- 2. Employees with no absences
SELECT e.emp_id, e.emp_name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1
    FROM attendance a
    WHERE a.emp_id = e.emp_id
    AND a.status = 'Absent'
);

-- 3. Department-wise average salary
SELECT d.dept_name, AVG(e.base_salary) avg_salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name;

-- 4. Department-wise average performance rating
SELECT d.dept_name, AVG(p.rating) avg_rating
FROM performance p
JOIN employees e ON p.emp_id = e.emp_id
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_name;

-- 5. Employees eligible for promotion
SELECT e.emp_id, e.emp_name
FROM employees e
JOIN performance p ON e.emp_id = p.emp_id
WHERE p.rating >= 4
AND e.hire_date <= CURDATE() - INTERVAL 3 YEAR;

-- 6. Salary increment using CTE
WITH increments AS (
    SELECT emp_id,
           CASE
             WHEN rating = 5 THEN 0.20
             WHEN rating = 4 THEN 0.10
             ELSE 0
           END inc_rate
    FROM performance
    WHERE review_year = 2025
)
SELECT e.emp_name,
       e.base_salary,
       e.base_salary + (e.base_salary * i.inc_rate) AS new_salary
FROM employees e
JOIN increments i ON e.emp_id = i.emp_id;

/* ================================
   TRIGGER
================================ */

DELIMITER $$

CREATE TRIGGER calc_net_salary
BEFORE INSERT ON payroll
FOR EACH ROW
BEGIN
    SET NEW.net_salary =
        (SELECT base_salary FROM employees WHERE emp_id = NEW.emp_id)
        + NEW.bonus - NEW.deductions;
END $$

DELIMITER ;

/* ================================
   STORED PROCEDURE
================================ */

DELIMITER $$

CREATE PROCEDURE generate_payroll(IN p_month DATE)
BEGIN
    INSERT INTO payroll (emp_id, pay_month, bonus, deductions, net_salary)
    SELECT emp_id, p_month, 5000, 2000, 0
    FROM employees;
END $$

DELIMITER ;

/* ================================
   INDEXES (OPTIMIZATION)
================================ */

CREATE INDEX idx_emp_dept ON employees(dept_id);
CREATE INDEX idx_att_date ON attendance(att_date);

/* ================================
   PROCEDURE EXECUTION
================================ */

CALL generate_payroll('2025-01-01');

/* ================================
   VIEW PAYROLL DATA
================================ */

SELECT * FROM payroll;
