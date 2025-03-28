---------------------------------------------------------------------------
**Lecturer:**  
**Group: 228 (235## Heading)**       
**Lab User:**  

**Student: Benjamin Purchase**     **NIA: 100557337**  
**Student: Ishaan Kalra**     **NIA: 100559793**  
**Student Lucas Pedemonte:**     **NIA: 100558486**  
---------------------------------------------------------------------------

# Introduction

It will consist of an introductory paragraph, dealing with the problem to be solved, analyzing it and fixing the goals of the work. It will also include the SQL code generated for this *labwork* (*queries*, *views,* *triggers*, ...). Finally, it should describe the document structure.

# Queries


#### **1.1 a) BoreBooks: Books with Editions in ≥3 Languages and No Loans**  

```
1. BookEditions := π(title, author, language) (books ⋈ editions)
2. LanguageCount := γ(title, author); COUNT(DISTINCT language)→lang_count (BookEditions)
3. QualifiedBooks := σ(lang_count ≥ 3) (LanguageCount)
4. LoanedBooks := π(title, author) (loans ⋈ copies ⋈ editions ⋈ books)
5. BoreBooks := QualifiedBooks - LoanedBooks
```

##### **Key operators:**
- **⋈** : Natural join (combines tables by matching column names)
- **π** : Projection (selects specific columns)
- **γ** : Group-by aggregation
- **σ** : Selection (filters rows)
- **-** : Set difference (finds records in first set but not second)

##### **How it works**:
1. First joins books with their editions
2. Groups by book + author to count distinct languages
3. Filters for books with ≥3 language editions
4. Finds all books that have been loaned
5. Returns only books that qualify (≥3 languages that haven't been loaned)

##### **SQL Implementation**  
```sql
WITH BookEditions AS (
    SELECT b.title, b.author, e.language, e.isbn
    FROM books b
    JOIN editions e ON b.title = e.title AND b.author = e.author
),
LanguageCount AS (
    SELECT title, author
    FROM BookEditions
    GROUP BY title, author
    HAVING COUNT(DISTINCT language) >= 3
),
LoanedBooks AS (
    SELECT DISTINCT b.title, b.author
    FROM copies c
    JOIN loans l ON c.signature = l.signature
    JOIN editions e ON c.isbn = e.isbn
    JOIN books b ON e.title = b.title AND e.author = b.author
)
SELECT lc.title, lc.author
FROM LanguageCount lc
LEFT JOIN LoanedBooks lb ON lc.title = lb.title AND lc.author = lb.author
WHERE lb.title IS NULL;
```

##### **Testing**  
1. **Test Case 1**:  
   - **Scenario**: Insert a book (*"Cézanne"* by *Cézanne, Paul*) with editions in Spanish, English, and French. No copies are loaned.  
   - **Expected Result**: The book appears in the output.  
   - **Actual Result**: *"Cézanne"* is listed in the query output (first row).  

2. **Test Case 2**:  
   - **Scenario**: A book (*"Costa Brava"* by *CampaÃ±Ã¡, Antonio*) with editions in 3 languages, but three copies are loaned.  
   - **Expected Result**: The book is excluded.  
   - **Actual Result**: *"Costa Brava"* does **not** appear in the output.  

3. **Test Case 3**:  
   - **Scenario**: A book (*"Abelardo Morell"* by *Morell, Abelardo*) has editions in only 2 languages with no copies loaned.  
   - **Expected Result**: Not included.  
   - **Actual Result**: *"Abelardo Morell"* is **not** in the output.  

This shows that the query correctly filters only the books with ≥3 languages and no loans.  

---

#### **1.1 b) Reports on Employees: Driver Metrics**  

##### **Relational Algebra**  

```
1. DriverInfo := π(passport, fullname, birthdate, cont_start, cont_end) (drivers)
2. Seniority := γ(passport); FLOOR(MONTHS_BETWEEN(cont_end, cont_start)/12)→seniority (DriverInfo)
3. ActiveYears := γ(passport); COUNT(DISTINCT YEAR(taskdate))→active_years (assign_drv)
4. StopsPerDriver := γ(passport); COUNT(town)→total_stops (assign_drv ⋈ stops)
5. LoansPerDriver := γ(passport); 
                      COUNT(signature)→total_loans,
                      SUM(CASE WHEN return IS NULL THEN 1 ELSE 0)→unreturned (services ⋈ loans)
6. FinalReport := DriverInfo ⋈ Seniority ⋈ ActiveYears ⋈ StopsPerDriver ⋈ LoansPerDriver
```

##### **Key steps:
1. **DriverInfo**: Gets driver's personal data
2. **Seniority**: Calculates contracted years (FLOOR rounds down)
3. **ActiveYears**: Counts distinct years with assignments
4. **StopsPerDriver**: Counts all stops made by each driver
5. **LoansPerDriver**: Calculates:
   - Total loans handled
   - Count of unreturned loans (where return IS NULL)
6. **FinalReport**: Joins all metrics together

##### **Why it works**:
- Each γ (gamma) operation groups by passport and calculates one metric
- The final join combines all metrics for the complete report
- Handles edge cases (like division by zero) through SQL implementation
- Matches your output showing drivers with their:
  - Age (from birthdate)
  - Seniority (contract duration) 
  - Stops/Year (total_stops/active_years)
  - % Unreturned (unreturned/total_loans)

This explains why see values like the following are shown:
- "109 Stops/Year" (109 stops ÷ 1 active year)
- "0% Unreturned" (when unreturned=0) in the results

##### **SQL Implementation**  
```sql
WITH DriverInfo AS (
    SELECT 
        passport, 
        fullname, 
        birthdate, 
        cont_start, 
        COALESCE(cont_end, SYSDATE) AS cont_end
    FROM drivers
),
Seniority AS (
    SELECT 
        passport, 
        FLOOR(MONTHS_BETWEEN(cont_end, cont_start) / 12) AS seniority 
    FROM DriverInfo
),
ActiveYears AS (
    SELECT 
        passport, 
        COUNT(DISTINCT EXTRACT(YEAR FROM taskdate)) AS active_years
    FROM assign_drv
    GROUP BY passport
),
StopsPerDriver AS (
    SELECT 
        ad.passport, 
        COUNT(s.town) AS total_stops
    FROM assign_drv ad
    JOIN stops s ON ad.route_id = s.route_id
    GROUP BY ad.passport
),
LoansPerDriver AS (
    SELECT 
        s.passport, 
        COUNT(l.signature) AS total_loans,
        SUM(CASE WHEN l.return IS NULL THEN 1 ELSE 0 END) AS unreturned
    FROM services s
    JOIN loans l ON s.town = l.town AND s.province = l.province AND s.taskdate = l.stopdate
    GROUP BY s.passport
)
SELECT 
    d.fullname AS "Full Name",
    FLOOR(MONTHS_BETWEEN(SYSDATE, d.birthdate) / 12) AS "Age",  
    s.seniority AS "Seniority (Years)",
    ay.active_years AS "Active Years",
    NVL(sp.total_stops / NULLIF(ay.active_years, 0), 0) AS "Stops/Year",
    NVL(lp.total_loans / NULLIF(ay.active_years, 0), 0) AS "Loans/Year",
    NVL((lp.unreturned / NULLIF(lp.total_loans, 0)) * 100, 0) AS "% Unreturned"
FROM DriverInfo d
JOIN Seniority s ON d.passport = s.passport
JOIN ActiveYears ay ON d.passport = ay.passport
LEFT JOIN StopsPerDriver sp ON d.passport = sp.passport
LEFT JOIN LoansPerDriver lp ON d.passport = lp.passport;
```

##### **Testing**  
1. **Test Case 1**:  
   - **Scenario**: Driver *"Victoria Rojo Blanco"* has:  
     - Age: 38, Seniority: 2 years, Active Years: 1  
     - Total Stops: 109, Total Loans: 1664, Unreturned: 0  
   - **Expected**:  
     - `Stops/Year = 109`, `Loans/Year = 1664`, `% Unreturned = 0`  
   - **Actual Result**: Matches output.  

2. **Test Case 2**:  
   - **Scenario**: Driver *"César Pareja Feliz"* has:  
     - Active Years: 3, Total Loans: 0  
   - **Expected**:  
     - `Loans/Year = 0`, `% Unreturned = 0`  
   - **Actual Result**: `Loans/Year = 0` (as shown in the output).  

3. **Test Case 3**:  
   - **Edge Case**: Driver with `active_years = 0` (e.g., no assignments).  
   - **Expected**: `Stops/Year = 0`, `Loans/Year = 0` (handled by `NULLIF`).  
   - **Result**: No such driver in the database.  

This shows that the query accurately calculates driver metrics, handles division by zero, and aligns with the provided results.  


# Package

Include an introduction with the structure of the package, and a subsection for each procedure or function that it includes. For each procedure, you must describe:

a) its design (inputs, outputs, logic of the main block), and in case of having needed to make use of auxiliary elements (queries, views, other procedures/functions...) their design and implementation must also be included (unless they are trivial queries).

b) its implementation in SQL

c) tests

# External Design

Describe the views and carry out their design, implementation, and tests (in a similar way to how the queries were made in section 2 but developing their operativity completeness where required). Include a subsection for each view you develop, outlining:

a) its design in relational algebra  
b) its implementation in SQL  
c) Tests: notice that it must be checked that the view is properly defined (like a query), as well as the operativity of the read and write views: it is necessary to establish which operations (insertion/deletion/modification) the manager resolves itself, and which other operations it does not.

> Operations on views not automatically supported by the manager must be resolved using triggers (of type *instead of*), which must also be described, implemented and tested in this section.

# Explicitly required Triggers

For each resolved trigger, include a subsection containing:

a) Description of the design: Table to which it is associated, Event or events in which it is triggered, Temporality (before, after or instead of), Granularity (by row or statement), Condition (if it has one) and Action (description in natural language).

b) Code (PL/SQL)

c) Tests

# Concluding Remarks

Firstly, you have to defend the achieved result, emphasizing the goodness of the semantic coverage, usage (comment unfeasible queries, in case), documentation, etc.

After stating your results, comment your achievement through this labwork: required effort (how much time you spent), knowledge gain, progress, etc. You can also propose improvements for further editions (size of the problem, requested items, deadlines, supporting materials, etc.).
