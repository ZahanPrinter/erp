# School ERP

A desktop school management application written in Java (Swing) on top of MySQL. Student, teacher, course and attendance records live in four relational tables. The UI never touches SQL: every table has its own **DAO class** that owns all of its queries, and every screen talks to the database only through those classes.

| Task                                       | Handled by                          | Underlying query                         |
|--------------------------------------------|-------------------------------------|------------------------------------------|
| Add, list, delete students / teachers      | `StudentDAO`, `TeacherDAO`          | `INSERT` / `SELECT *` / `DELETE`         |
| Add, list, delete courses                  | `CourseDAO`                         | `INSERT` / `SELECT *` / `DELETE`         |
| Mark attendance for a student on a date    | `AttendanceDAO.markAttendance`      | `INSERT`                                 |
| View today's or all attendance             | `AttendanceDAO`                     | `JOIN students`, `WHERE date = today`    |
| Dashboard totals                           | `get*Count()` on each DAO           | `SELECT COUNT(*)`                        |

## Features

- Add, view and delete students, teachers and courses
- Attendance marking with a built-in calendar date picker (Present / Absent / Late)
- Attendance views for today only or for all records, joined with student names
- Input validation on every form: emails must match a standard address pattern and contain `@`, phone numbers must be exactly 10 digits, required fields cannot be empty
- Dashboard with live counts of students, teachers and courses
- Fully custom dark theme: buttons, tables, text fields, combo boxes, scroll panes and the date picker are all custom-painted components, so the look does not depend on the platform's default Swing theme
- Tables are created automatically on first run
- A MySQL script that fills the database with realistic fake data for testing

## How it works

`DatabaseConnection` opens a single JDBC connection, creates the four tables if they do not exist, and hands the connection to each DAO. Every DAO uses `PreparedStatement` for anything that takes input, and returns plain model objects (`Student`, `Teacher`, `Course`, `Attendance`) so the UI never sees a `ResultSet`.

**Schema:** four tables with foreign keys.

```
students   (id, name, roll_no*, class, email, phone, address, admission_date)
teachers   (id, name, employee_id*, subject, email, phone, qualification, join_date)
courses    (id, course_name, course_code*, teacher_id -> teachers.id, credits)
attendance (id, student_id -> students.id, date, status ENUM('Present','Absent','Late'))
```

`*` marks a unique column. Deleting a teacher sets `courses.teacher_id` to `NULL`. Deleting a student deletes that student's attendance rows (`ON DELETE CASCADE`).

**UI:** `MainFrame` holds a sidebar and a `CardLayout`; each sidebar button switches to one panel (`DashboardPanel`, `StudentPanel`, `TeacherPanel`, `CoursePanel`, `AttendancePanel`). The list panels share the same layout: a `DarkTable` inside a `DarkScrollPane`, with Add, Refresh and Delete buttons.

**Theme:** `DarkTheme.apply()` runs before any window is created and overrides the Swing defaults. The `Dark*` components then paint their own backgrounds, which avoids white buttons and table headers caused by the system look and feel.

**Date picker:** `DarkDatePicker` renders a month grid and fires a listener when a day is clicked. It has previous / next month buttons and a Today shortcut. The attendance dialog reads the chosen `LocalDate` and converts it to `java.sql.Date`.

**Validation:** `ValidationUtils` holds the email and phone regexes plus `validateStudentForm` and `validateTeacherForm`. Forms call these before the DAO is touched, so invalid data never reaches the database.

## Project structure

```
SchoolERP/
├── src/
│   ├── main/
│   │   └── Main.java                 # entry point, applies theme, opens MainFrame
│   ├── database/
│   │   └── DatabaseConnection.java   # JDBC connection + table creation
│   ├── models/
│   │   ├── Student.java
│   │   ├── Teacher.java
│   │   ├── Course.java
│   │   └── Attendance.java
│   ├── dao/
│   │   ├── StudentDAO.java
│   │   ├── TeacherDAO.java
│   │   ├── CourseDAO.java
│   │   └── AttendanceDAO.java
│   ├── utils/
│   │   ├── UIConstants.java          # colors and fonts
│   │   ├── DarkTheme.java            # global Swing overrides
│   │   └── ValidationUtils.java      # email / phone / required-field checks
│   └── ui/
│       ├── MainFrame.java            # window, sidebar, card navigation
│       ├── components/
│       │   ├── DarkButton.java
│       │   ├── DarkTextField.java
│       │   ├── DarkComboBox.java
│       │   ├── DarkTable.java
│       │   ├── DarkScrollPane.java
│       │   ├── DarkDatePicker.java
│       │   └── StatCard.java
│       └── panels/
│           ├── DashboardPanel.java
│           ├── StudentPanel.java
│           ├── TeacherPanel.java
│           ├── CoursePanel.java
│           └── AttendancePanel.java
├── sql/
│   └── generate_fake_data.sql        # bulk test data
├── lib/
│   └── mysql-connector-j-8.x.x.jar
└── README.md
```

## Building and running

Requires JDK 17 or newer, MySQL 8.0.13 or newer (the schema uses `DEFAULT (CURRENT_DATE)`), and the MySQL Connector/J jar in `lib/`.

1. Create the database:

```sql
CREATE DATABASE school_erp;
```

2. Set your credentials in `src/database/DatabaseConnection.java`:

```java
private static final String URL = "jdbc:mysql://localhost:3306/school_erp";
private static final String USER = "root";
private static final String PASSWORD = "your_password";
```

3. Compile and run from the project root.

Linux / macOS:

```
javac -cp "lib/*" -d out $(find src -name "*.java")
java -cp "out:lib/*" main.Main
```

Windows (PowerShell):

```
javac -cp "lib/*" -d out (Get-ChildItem -Recurse src -Filter *.java).FullName
java -cp "out;lib/*" main.Main
```

The classpath separator is `:` on Linux and macOS and `;` on Windows. In VS Code, add the jar under **Referenced Libraries** and run `Main.java`.

## Usage

The sidebar switches between five screens:

```
Dashboard    totals for students, teachers, courses
Students     table + Add New / Refresh / Delete
Teachers     table + Add New / Refresh / Delete
Courses      table + Add New / Refresh / Delete
Attendance   table + Mark Attendance / Today's / All Records / Delete
```

To mark attendance, click **Mark Attendance**, pick a day on the calendar, choose the student and status, and save. The table then reloads with today's records.

### Validation rules

| Field            | Rule                                                        |
|------------------|-------------------------------------------------------------|
| Email            | Must contain `@` and match `name@domain.tld`                |
| Phone            | Exactly 10 digits, no spaces, dashes or letters             |
| Name, ID, class  | Required (student: roll no and class, teacher: employee ID and subject) |

## Sample data

`sql/generate_fake_data.sql` creates temporary stored procedures that fill the database in this order, because later tables reference earlier ones:

| Table      | Rows                                              |
|------------|---------------------------------------------------|
| teachers   | 500                                               |
| courses    | 100                                               |
| students   | 10,000                                            |
| attendance | about 220,000 (last 30 days, weekends skipped)    |

Attendance is weighted 85% Present, 10% Absent, 5% Late. Generated phone numbers have 10 digits and emails contain `@`, so they pass the app's own validation. Run it from the command line, which tolerates long scripts better than GUI clients:

```
mysql -u root -p --connect-timeout=600 school_erp < sql/generate_fake_data.sql
```

The script commits in batches and runs `SET GLOBAL` for `wait_timeout` and the network timeouts, which needs an account with the privilege to do so. Those changes last until the MySQL server restarts. The procedures and temporary tables are dropped when it finishes.
