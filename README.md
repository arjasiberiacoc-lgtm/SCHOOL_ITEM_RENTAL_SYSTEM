# SCHOOL_ITEM_RENTAL_SYSTEM
package com.mycompany.rentalsystem.persistence;

import com.mycompany.rentalsystem.model.Item;
import com.mycompany.rentalsystem.model.Rental;
import com.mycompany.rentalsystem.model.UserAccount;
import com.mycompany.rentalsystem.model.YearLevel;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.sql.Timestamp;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

public class MySqlRentalRepository {
    private static final String DEFAULT_URL =
            "jdbc:mysql://localhost:3306/school_item_rental?createDatabaseIfNotExist=true&serverTimezone=UTC";
    private final String url = setting("DB_URL", DEFAULT_URL);
    private final String username = setting("DB_USER", "root");
    private final String password = setting("DB_PASSWORD", "");

    public MySqlRentalRepository() throws SQLException {
        initializeSchema();
        seedItemsIfEmpty();
    }

    private static String setting(String name, String fallback) {
        String value = System.getenv(name);
        return value == null || value.isBlank() ? fallback : value;
    }

    private Connection connect() throws SQLException {
        return DriverManager.getConnection(url, username, password);
    }

    private void initializeSchema() throws SQLException {
        try (Connection connection = connect(); Statement statement = connection.createStatement()) {
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS departments (
                        department_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                        name VARCHAR(100) NOT NULL UNIQUE
                    ) ENGINE=InnoDB
                    """);
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS year_levels (
                        year_level_id INT UNSIGNED PRIMARY KEY,
                        name VARCHAR(40) NULL
                    ) ENGINE=InnoDB
                    """);
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS categories (
                        category_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                        name VARCHAR(100) NOT NULL UNIQUE
                    ) ENGINE=InnoDB
                    """);
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS roles (
                        role_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                        name VARCHAR(20) NOT NULL UNIQUE
                    ) ENGINE=InnoDB
                    """);
            ensureYearLevelNameColumn(connection);
            seedLookupTables(connection);
            seedRoles(connection);
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS students (
                        student_key INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                        student_id VARCHAR(30) NOT NULL UNIQUE,
                        first_name VARCHAR(120) NOT NULL,
                        last_name VARCHAR(120) NOT NULL,
                        department_id INT UNSIGNED NULL,
                        year_level_id INT UNSIGNED NULL,
                        user_id INT UNSIGNED NULL,
                        UNIQUE KEY uq_students_user_id (user_id),
                        CONSTRAINT fk_students_department FOREIGN KEY (department_id)
                            REFERENCES departments(department_id) ON DELETE RESTRICT,
                        CONSTRAINT fk_students_year_level FOREIGN KEY (year_level_id)
                            REFERENCES year_levels(year_level_id) ON DELETE RESTRICT
                    ) ENGINE=InnoDB
                    """);
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS users (
                        user_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                        username VARCHAR(80) NOT NULL UNIQUE,
                        password_hash VARCHAR(255) NOT NULL,
                        role_id INT UNSIGNED NOT NULL,
                        created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                        CONSTRAINT fk_users_role FOREIGN KEY (role_id)
                            REFERENCES roles(role_id) ON DELETE RESTRICT
                    ) ENGINE=InnoDB
                    """);
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS items (
                        item_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                        name VARCHAR(120) NOT NULL,
                        category_id INT UNSIGNED NOT NULL,
                        rental_fee_cents BIGINT UNSIGNED NOT NULL,
                        quantity INT UNSIGNED NOT NULL DEFAULT 1,
                        is_active BOOLEAN NOT NULL DEFAULT TRUE,
                        UNIQUE KEY uq_items_name (name),
                        CONSTRAINT fk_items_category FOREIGN KEY (category_id)
                            REFERENCES categories(category_id) ON DELETE RESTRICT,
                        CHECK (rental_fee_cents >= 0),
                        CHECK (quantity > 0)
                    ) ENGINE=InnoDB
                    """);
            statement.executeUpdate("""
                    CREATE TABLE IF NOT EXISTS rentals (
                        rental_id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                        transaction_id INT UNSIGNED NOT NULL,
                        student_key INT UNSIGNED NOT NULL,
                        item_id INT UNSIGNED NOT NULL,
                        rental_fee_cents BIGINT UNSIGNED NOT NULL,
                        rented_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                        due_at DATETIME NOT NULL,
                        returned_at DATETIME NULL,
                        KEY rentals_by_item (item_id),
                        KEY rentals_by_student (student_key),
                        KEY rentals_by_due_date (due_at),
                        CONSTRAINT fk_rentals_student FOREIGN KEY (student_key)
                            REFERENCES students(student_key) ON DELETE RESTRICT,
                        CONSTRAINT fk_rentals_item FOREIGN KEY (item_id)
                            REFERENCES items(item_id) ON DELETE RESTRICT,
                        CHECK (rental_fee_cents >= 0),
                        CHECK (due_at >= rented_at)
                    ) ENGINE=InnoDB
                    """);
            migrateStudentSchema(connection);
            migrateStudentLookupColumns(connection);
            migrateAccountRoleAndStudentLink(connection);
            migrateStudentPrimaryKey(connection);
            migrateRentalStudentKey(connection);
            migrateQuantitySchema(connection);
            migrateItemCategoryColumn(connection);
            migrateItemNameConstraint(connection);
            migrateYearLevelNames(connection);
            migrateIdentifierTypes(connection);
        }
    }

    private void seedRoles(Connection connection) throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement(
                "INSERT IGNORE INTO roles (name) VALUES (?)")) {
            for (String role : new String[]{"ADMIN", "STUDENT"}) {
                statement.setString(1, role);
                statement.addBatch();
            }
            statement.executeBatch();
        }
    }

    private void seedLookupTables(Connection connection) throws SQLException {
        try (PreparedStatement department = connection.prepareStatement(
                "INSERT IGNORE INTO departments (name) VALUES (?)")) {
            String[] departments = {"BSIT", "BSCS", "BSBA", "BSA", "BSHM", "BSCRIM", "BSED", "BEED"};
            for (String name : departments) {
                department.setString(1, name);
                department.addBatch();
            }
            department.executeBatch();
        }
        try (PreparedStatement yearLevel = connection.prepareStatement(
                "INSERT IGNORE INTO year_levels (year_level_id, name) VALUES (?, ?)")) {
            String[] names = {"First Year", "Second Year", "Third Year", "Fourth Year"};
            for (int index = 0; index < names.length; index++) {
                yearLevel.setInt(1, index + 1);
                yearLevel.setString(2, names[index]);
                yearLevel.addBatch();
            }
            yearLevel.executeBatch();
        }
        try (PreparedStatement category = connection.prepareStatement(
                "INSERT IGNORE INTO categories (name) VALUES (?)")) {
            String[] categories = {
                "School Supplies", "Personal Items", "Electronics", "Laboratory Equipment", "Other"
            };
            for (String name : categories) {
                category.setString(1, name);
                category.addBatch();
            }
            category.executeBatch();
        }
    }

    private void ensureYearLevelNameColumn(Connection connection) throws SQLException {
        if (!hasColumn(connection, "year_levels", "name")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE year_levels ADD COLUMN name VARCHAR(40) NULL");
        }
    }

    private void migrateYearLevelNames(Connection connection) throws SQLException {
        connection.createStatement().executeUpdate("""
                UPDATE year_levels
                SET name = CASE year_level_id
                    WHEN 1 THEN 'First Year'
                    WHEN 2 THEN 'Second Year'
                    WHEN 3 THEN 'Third Year'
                    WHEN 4 THEN 'Fourth Year'
                    ELSE CONCAT('Year ', year_level_id)
                END
                WHERE name IS NULL OR TRIM(name) = ''
                """);
        if (isNullableColumn(connection, "year_levels", "name")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE year_levels MODIFY name VARCHAR(40) NOT NULL");
        }
        if (!hasIndex(connection, "year_levels", "uq_year_levels_name")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE year_levels ADD UNIQUE KEY uq_year_levels_name (name)");
        }
    }

    private void migrateStudentSchema(Connection connection) throws SQLException {
        boolean hasFirstName = hasColumn(connection, "students", "first_name");
        boolean hasLastName = hasColumn(connection, "students", "last_name");
        boolean hasFullName = hasColumn(connection, "students", "full_name");

        if (!hasFirstName) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD COLUMN first_name VARCHAR(120) NULL");
        }

        if (!hasLastName) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD COLUMN last_name VARCHAR(120) NULL");
        }
        if (hasFullName) {
            connection.createStatement().executeUpdate("""
                    UPDATE students
                    SET first_name = TRIM(SUBSTRING_INDEX(TRIM(full_name), ' ', 1)),
                        last_name = CASE
                            WHEN LOCATE(' ', TRIM(full_name)) = 0 THEN ''
                            ELSE TRIM(SUBSTRING(TRIM(full_name), LOCATE(' ', TRIM(full_name)) + 1))
                        END
                    WHERE first_name IS NULL OR last_name IS NULL
                    """);
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students MODIFY first_name VARCHAR(120) NOT NULL");
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students MODIFY last_name VARCHAR(120) NOT NULL");
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students DROP COLUMN full_name");
        }
    }

    private void migrateStudentLookupColumns(Connection connection) throws SQLException {
        if (!hasColumn(connection, "students", "department_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD COLUMN department_id INT UNSIGNED NULL");
        }
        if (!hasColumn(connection, "students", "year_level_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD COLUMN year_level_id INT UNSIGNED NULL");
        }
        if (hasColumn(connection, "students", "department")) {
            connection.createStatement().executeUpdate("""
                    INSERT IGNORE INTO departments (name)
                    SELECT DISTINCT TRIM(department) FROM students
                    WHERE department IS NOT NULL AND TRIM(department) <> ''
                    """);
            connection.createStatement().executeUpdate("""
                    UPDATE students s
                    JOIN departments d ON d.name = TRIM(s.department)
                    SET s.department_id = d.department_id
                    WHERE s.department_id IS NULL AND s.department IS NOT NULL
                      AND TRIM(s.department) <> ''
                    """);
            connection.createStatement().executeUpdate("ALTER TABLE students DROP COLUMN department");
        }
        if (hasColumn(connection, "students", "year_level")) {
            connection.createStatement().executeUpdate("""
                    INSERT IGNORE INTO year_levels (year_level_id, name)
                    SELECT DISTINCT year_level,
                        CASE year_level
                            WHEN 1 THEN 'First Year'
                            WHEN 2 THEN 'Second Year'
                            WHEN 3 THEN 'Third Year'
                            WHEN 4 THEN 'Fourth Year'
                            ELSE CONCAT('Year ', year_level)
                        END
                    FROM students WHERE year_level IS NOT NULL
                    """);
            connection.createStatement().executeUpdate("""
                    UPDATE students
                    SET year_level_id = year_level
                    WHERE year_level_id IS NULL AND year_level IS NOT NULL
                    """);
            connection.createStatement().executeUpdate("ALTER TABLE students DROP COLUMN year_level");
        }
        addForeignKey(connection, "students", "fk_students_department",
                "FOREIGN KEY (department_id) REFERENCES departments(department_id) ON DELETE RESTRICT");
        addForeignKey(connection, "students", "fk_students_year_level",
                "FOREIGN KEY (year_level_id) REFERENCES year_levels(year_level_id) ON DELETE RESTRICT");
    }

    private void migrateItemCategoryColumn(Connection connection) throws SQLException {
        if (!hasColumn(connection, "items", "category_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE items ADD COLUMN category_id INT UNSIGNED NULL");
        }
        connection.createStatement().executeUpdate("""
                UPDATE items i
                JOIN categories c ON c.name = CASE
                    WHEN LOWER(i.name) = 'calculator' THEN 'School Supplies'
                    WHEN LOWER(i.name) = 'umbrella' THEN 'Personal Items'
                    WHEN LOWER(i.name) = 'charger' THEN 'Electronics'
                    WHEN LOWER(i.name) = 'lab equipment' THEN 'Laboratory Equipment'
                    ELSE 'Other'
                END
                SET i.category_id = c.category_id
                WHERE i.category_id IS NULL
                """);
        if (isNullableColumn(connection, "items", "category_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE items MODIFY category_id INT UNSIGNED NOT NULL");
        }
        addForeignKey(connection, "items", "fk_items_category",
                "FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE RESTRICT");
    }

    private void migrateAccountRoleAndStudentLink(Connection connection) throws SQLException {
        if (!hasColumn(connection, "users", "role_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE users ADD COLUMN role_id INT UNSIGNED NULL");
        }
        if (hasColumn(connection, "users", "role")) {
            connection.createStatement().executeUpdate("""
                    UPDATE users u
                    JOIN roles r ON r.name = u.role
                    SET u.role_id = r.role_id
                    WHERE u.role_id IS NULL
                    """);
        }
        connection.createStatement().executeUpdate("""
                UPDATE users u
                JOIN roles r ON r.name = 'STUDENT'
                SET u.role_id = r.role_id
                WHERE u.role_id IS NULL
                """);
        if (isNullableColumn(connection, "users", "role_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE users MODIFY role_id INT UNSIGNED NOT NULL");
        }
        addForeignKey(connection, "users", "fk_users_role",
                "FOREIGN KEY (role_id) REFERENCES roles(role_id) ON DELETE RESTRICT");

        if (!hasColumn(connection, "students", "user_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD COLUMN user_id INT UNSIGNED NULL");
        }
        if (hasColumn(connection, "users", "student_id")) {
            connection.createStatement().executeUpdate("""
                    UPDATE students s
                    JOIN users u ON u.student_id = s.student_id
                    SET s.user_id = u.user_id
                    WHERE s.user_id IS NULL
                    """);
            dropConstraintsUsingColumn(connection, "users", "student_id");
        }
        dropCheckConstraintsReferencingColumn(connection, "users", "student_id");
        if (hasColumn(connection, "users", "student_id")) {
            connection.createStatement().executeUpdate("ALTER TABLE users DROP COLUMN student_id");
        }
        if (hasColumn(connection, "users", "role")) {
            connection.createStatement().executeUpdate("ALTER TABLE users DROP COLUMN role");
        }
        if (!hasIndex(connection, "students", "uq_students_user_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD UNIQUE KEY uq_students_user_id (user_id)");
        }
        addForeignKey(connection, "students", "fk_students_user",
                "FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE SET NULL");
    }

    private void migrateIdentifierTypes(Connection connection) throws SQLException {
        dropConstraintsUsingColumn(connection, "students", "year_level_id");
        dropConstraintsUsingColumn(connection, "students", "user_id");
        dropConstraintsUsingColumn(connection, "users", "role_id");

        try (Statement statement = connection.createStatement()) {
            statement.executeUpdate("ALTER TABLE year_levels MODIFY year_level_id INT UNSIGNED NOT NULL");
            statement.executeUpdate("ALTER TABLE students MODIFY year_level_id INT UNSIGNED NULL");
            statement.executeUpdate("ALTER TABLE roles MODIFY role_id INT UNSIGNED NOT NULL AUTO_INCREMENT");
            statement.executeUpdate("ALTER TABLE users MODIFY user_id INT UNSIGNED NOT NULL AUTO_INCREMENT");
            statement.executeUpdate("ALTER TABLE users MODIFY role_id INT UNSIGNED NOT NULL");
            statement.executeUpdate("ALTER TABLE students MODIFY user_id INT UNSIGNED NULL");
            statement.executeUpdate("ALTER TABLE rentals MODIFY rental_id INT UNSIGNED NOT NULL AUTO_INCREMENT");
            statement.executeUpdate("ALTER TABLE rentals MODIFY transaction_id INT UNSIGNED NOT NULL");
        }
        addForeignKey(connection, "students", "fk_students_year_level",
                "FOREIGN KEY (year_level_id) REFERENCES year_levels(year_level_id) ON DELETE RESTRICT");
        addForeignKey(connection, "students", "fk_students_user",
                "FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE SET NULL");
        addForeignKey(connection, "users", "fk_users_role",
                "FOREIGN KEY (role_id) REFERENCES roles(role_id) ON DELETE RESTRICT");
    }

    private void migrateStudentPrimaryKey(Connection connection) throws SQLException {
        dropConstraintsUsingColumn(connection, "rentals", "student_id");
        if (!hasColumn(connection, "students", "student_key")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD COLUMN student_key INT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE");
        }
        if (!isPrimaryKeyColumn(connection, "students", "student_key")) {
            if (hasIndex(connection, "students", "PRIMARY")) {
                connection.createStatement().executeUpdate("ALTER TABLE students DROP PRIMARY KEY");
            }
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD PRIMARY KEY (student_key)");
        }
        if (!hasIndex(connection, "students", "uq_students_school_id")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE students ADD UNIQUE KEY uq_students_school_id (student_id)");
        }
    }

    private void migrateRentalStudentKey(Connection connection) throws SQLException {
        boolean hasStudentId = hasColumn(connection, "rentals", "student_id");
        if (!hasColumn(connection, "rentals", "student_key")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE rentals ADD COLUMN student_key INT UNSIGNED NULL");
        }
        if (hasStudentId) {
            connection.createStatement().executeUpdate("""
                    UPDATE rentals r
                    JOIN students s ON s.student_id = r.student_id
                    SET r.student_key = s.student_key
                    WHERE r.student_key IS NULL
                    """);
            dropConstraintsUsingColumn(connection, "rentals", "student_id");
            if (hasIndex(connection, "rentals", "rentals_by_student")) {
                connection.createStatement().executeUpdate(
                        "ALTER TABLE rentals DROP INDEX rentals_by_student");
            }
            connection.createStatement().executeUpdate("ALTER TABLE rentals DROP COLUMN student_id");
        }
        try (Statement statement = connection.createStatement();
                ResultSet result = statement.executeQuery(
                        "SELECT COUNT(*) FROM rentals WHERE student_key IS NULL")) {
            result.next();
            if (result.getLong(1) != 0) {
                throw new SQLException("Some rental records could not be linked to a student profile.");
            }
        }
        connection.createStatement().executeUpdate(
                "ALTER TABLE rentals MODIFY student_key INT UNSIGNED NOT NULL");
        if (!hasIndex(connection, "rentals", "rentals_by_student")) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE rentals ADD KEY rentals_by_student (student_key)");
        }
        addForeignKey(connection, "rentals", "fk_rentals_student",
                "FOREIGN KEY (student_key) REFERENCES students(student_key) ON DELETE RESTRICT");
        }

    private void dropConstraintsUsingColumn(Connection connection, String tableName, String columnName)
            throws SQLException {
        List<String> constraints = new ArrayList<>();
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT DISTINCT constraint_name
                FROM information_schema.key_column_usage
                WHERE table_schema = DATABASE() AND table_name = ? AND column_name = ?
                  AND referenced_table_name IS NOT NULL
                """)) {
            statement.setString(1, tableName);
            statement.setString(2, columnName);
            try (ResultSet result = statement.executeQuery()) {
                while (result.next()) {
                    constraints.add(result.getString(1));
                }
            }
        }
        for (String constraint : constraints) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE " + tableName + " DROP FOREIGN KEY " + constraint);
        }
    }

    private void dropCheckConstraintsReferencingColumn(Connection connection, String tableName, String columnName)
            throws SQLException {
        List<String> constraints = new ArrayList<>();
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT tc.constraint_name
                FROM information_schema.table_constraints tc
                JOIN information_schema.check_constraints cc
                  ON cc.constraint_schema = tc.constraint_schema
                 AND cc.constraint_name = tc.constraint_name
                WHERE tc.constraint_schema = DATABASE() AND tc.table_name = ?
                  AND tc.constraint_type = 'CHECK'
                  AND cc.check_clause LIKE ?
                """)) {
            statement.setString(1, tableName);
            statement.setString(2, "%" + columnName + "%");
            try (ResultSet result = statement.executeQuery()) {
                while (result.next()) {
                    constraints.add(result.getString(1));
                }
            }
        }
        for (String constraint : constraints) {
            connection.createStatement().executeUpdate(
                    "ALTER TABLE " + tableName + " DROP CHECK " + constraint);
        }
    }

    private boolean hasIndex(Connection connection, String tableName, String indexName) throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.statistics
                WHERE table_schema = DATABASE() AND table_name = ? AND index_name = ?
                """)) {
            statement.setString(1, tableName);
            statement.setString(2, indexName);
            try (ResultSet result = statement.executeQuery()) {
                result.next();
                return result.getInt(1) > 0;
            }
        }
    }

    private boolean isPrimaryKeyColumn(Connection connection, String tableName, String columnName)
            throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.key_column_usage
                WHERE table_schema = DATABASE() AND table_name = ?
                  AND constraint_name = 'PRIMARY' AND column_name = ?
                """)) {
            statement.setString(1, tableName);
            statement.setString(2, columnName);
            try (ResultSet result = statement.executeQuery()) {
                result.next();
                return result.getInt(1) != 0;
            }
        }
    }

    private void addForeignKey(Connection connection, String tableName, String constraintName, String definition)
            throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.table_constraints
                WHERE constraint_schema = DATABASE() AND table_name = ? AND constraint_name = ?
                """)) {
            statement.setString(1, tableName);
            statement.setString(2, constraintName);
            try (ResultSet result = statement.executeQuery()) {
                result.next();
                if (result.getInt(1) == 0) {
                    connection.createStatement().executeUpdate(
                            "ALTER TABLE " + tableName + " ADD CONSTRAINT " + constraintName + " " + definition);
                }
            }
        }
    }

    private boolean hasColumn(Connection connection, String tableName, String columnName) throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.columns
                WHERE table_schema = DATABASE() AND table_name = ? AND column_name = ?
                """)) {
            statement.setString(1, tableName);
            statement.setString(2, columnName);
            try (ResultSet result = statement.executeQuery()) {
                result.next();
                return result.getInt(1) > 0;
            }
        }
    }

    private boolean isNullableColumn(Connection connection, String tableName, String columnName) throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT is_nullable FROM information_schema.columns
                WHERE table_schema = DATABASE() AND table_name = ? AND column_name = ?
                """)) {
            statement.setString(1, tableName);
            statement.setString(2, columnName);
            try (ResultSet result = statement.executeQuery()) {
                return result.next() && "YES".equals(result.getString(1));
            }
        }
    }

    private void migrateItemNameConstraint(Connection connection) throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.statistics
                WHERE table_schema = DATABASE() AND table_name = 'items'
                  AND index_name = 'uq_items_name'
                """); ResultSet result = statement.executeQuery()) {
            result.next();
            if (result.getInt(1) == 0) {
                connection.createStatement().executeUpdate(
                        "ALTER TABLE items ADD UNIQUE KEY uq_items_name (name)");
            }
        }
    }

    private void migrateQuantitySchema(Connection connection) throws SQLException {
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.columns
                WHERE table_schema = DATABASE() AND table_name = 'items' AND column_name = 'quantity'
                """); ResultSet result = statement.executeQuery()) {
            result.next();
            if (result.getInt(1) == 0) {
                connection.createStatement().executeUpdate(
                        "ALTER TABLE items ADD COLUMN quantity INT UNSIGNED NOT NULL DEFAULT 1");
            }
        }
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.columns
                WHERE table_schema = DATABASE() AND table_name = 'rentals' AND column_name = 'transaction_id'
                """); ResultSet result = statement.executeQuery()) {
            result.next();
            if (result.getInt(1) == 0) {
                connection.createStatement().executeUpdate(
                        "ALTER TABLE rentals ADD COLUMN transaction_id BIGINT UNSIGNED NULL");
            }
        }
        try (PreparedStatement statement = connection.prepareStatement("""
                SELECT COUNT(*) FROM information_schema.statistics
                WHERE table_schema = DATABASE() AND table_name = 'rentals'
                  AND index_name = 'one_active_rental_per_item'
                """); ResultSet result = statement.executeQuery()) {
            result.next();
            if (result.getInt(1) > 0) {
                connection.createStatement().executeUpdate(
                        "ALTER TABLE rentals DROP INDEX one_active_rental_per_item");
            }
        }
    }

    private void seedItemsIfEmpty() throws SQLException {
        try (Connection connection = connect();
                Statement statement = connection.createStatement();
                ResultSet result = statement.executeQuery("SELECT COUNT(*) FROM items")) {
            result.next();
            if (result.getLong(1) != 0) {
                return;
            }
        }
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement("""
                INSERT INTO items (name, category_id, rental_fee_cents)
                SELECT ?, category_id, ? FROM categories WHERE name = ?
                """)) {
            String[] names = {"Calculator", "Umbrella", "Charger", "Lab Equipment"};
            String[] categories = {"School Supplies", "Personal Items", "Electronics", "Laboratory Equipment"};
            long[] fees = {2000, 1500, 2500, 5000};
            for (int index = 0; index < names.length; index++) {
                statement.setString(1, names[index]);
                statement.setLong(2, fees[index]);
                statement.setString(3, categories[index]);
                statement.addBatch();
            }
            statement.executeBatch();
        }
    }

    public List<String> getDepartments() throws SQLException {
        return getLookupValues("SELECT name FROM departments ORDER BY name");
    }

    public List<YearLevel> getYearLevels() throws SQLException {
        List<YearLevel> values = new ArrayList<>();
        try (Connection connection = connect();
                PreparedStatement statement = connection.prepareStatement(
                        "SELECT year_level_id, name FROM year_levels ORDER BY year_level_id");
                ResultSet result = statement.executeQuery()) {
            while (result.next()) {
                values.add(new YearLevel(result.getInt("year_level_id"), result.getString("name")));
            }
        }
        return values;
    }

    public List<String> getCategories() throws SQLException {
        return getLookupValues("SELECT name FROM categories ORDER BY name");
    }

    private List<String> getLookupValues(String sql) throws SQLException {
        List<String> values = new ArrayList<>();
        try (Connection connection = connect();
                PreparedStatement statement = connection.prepareStatement(sql);
                ResultSet result = statement.executeQuery()) {
            while (result.next()) {
                values.add(result.getString(1));
            }
        }
        return values;
    }

    public List<Item> getItems() throws SQLException {
        String sql = """
                SELECT i.item_id, i.name, c.name AS category, i.rental_fee_cents, i.quantity, i.is_active,
                       i.quantity - (
                           SELECT COUNT(*) FROM rentals r
                           WHERE r.item_id = i.item_id AND r.returned_at IS NULL
                       ) AS available_quantity
                FROM items i JOIN categories c ON c.category_id = i.category_id ORDER BY i.item_id
                """;
        List<Item> items = new ArrayList<>();
        try (Connection connection = connect();
                PreparedStatement statement = connection.prepareStatement(sql);
                ResultSet result = statement.executeQuery()) {
            while (result.next()) {
                Item item = new Item(result.getInt("item_id"), result.getString("name"),
                        result.getString("category"),
                        result.getLong("rental_fee_cents") / 100.0,
                        result.getInt("quantity"), result.getInt("available_quantity"));
                item.setActive(result.getBoolean("is_active"));
                items.add(item);
            }
        }
        return items;
    }

    public List<Rental> getRentals() throws SQLException {
        String sql = """
                  SELECT r.rental_id, COALESCE(r.transaction_id, r.rental_id) AS transaction_id,
                      s.student_id, CONCAT_WS(' ', s.first_name, s.last_name) AS full_name,
                      r.rental_fee_cents,
                       r.rented_at, r.due_at, r.returned_at,
                       i.item_id, i.name AS item_name, c.name AS item_category,
                       i.rental_fee_cents AS item_fee_cents,
                       i.quantity, i.is_active,
                       i.quantity - (
                           SELECT COUNT(*) FROM rentals active_rental
                           WHERE active_rental.item_id = i.item_id
                             AND active_rental.returned_at IS NULL
                       ) AS available_quantity
                FROM rentals r
                JOIN students s ON s.student_key = r.student_key
                JOIN items i ON i.item_id = r.item_id
                JOIN categories c ON c.category_id = i.category_id
                ORDER BY r.rented_at DESC, r.rental_id DESC
                """;
        List<Rental> rentals = new ArrayList<>();
        try (Connection connection = connect();
                PreparedStatement statement = connection.prepareStatement(sql);
                ResultSet result = statement.executeQuery()) {
            while (result.next()) {
                Item item = new Item(result.getInt("item_id"), result.getString("item_name"),
                        result.getString("item_category"),
                        result.getLong("item_fee_cents") / 100.0,
                        result.getInt("quantity"), result.getInt("available_quantity"));
                item.setActive(result.getBoolean("is_active"));
                Timestamp returnedTimestamp = result.getTimestamp("returned_at");
                LocalDateTime returnedAt = returnedTimestamp == null
                        ? null : returnedTimestamp.toLocalDateTime();
                rentals.add(new Rental(result.getLong("rental_id"), result.getLong("transaction_id"),
                    result.getString("student_id"),
                        result.getString("full_name"), item,
                        result.getLong("rental_fee_cents") / 100.0,
                        result.getTimestamp("rented_at").toLocalDateTime(),
                        result.getTimestamp("due_at").toLocalDateTime(), returnedAt));
            }
        }
        return rentals;
    }

    public int addItem(String name, String category, long feeCents, int quantity) throws SQLException {
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement("""
                INSERT INTO items (name, category_id, rental_fee_cents, quantity)
                SELECT ?, category_id, ?, ? FROM categories WHERE name = ?
                """,
                Statement.RETURN_GENERATED_KEYS)) {
            statement.setString(1, name);
            statement.setLong(2, feeCents);
            statement.setInt(3, quantity);
            statement.setString(4, category);
            if (statement.executeUpdate() != 1) {
                throw new SQLException("The selected item category no longer exists.");
            }
            try (ResultSet keys = statement.getGeneratedKeys()) {
                if (!keys.next()) {
                    throw new SQLException("The database did not return the new item ID.");
                }
                return keys.getInt(1);
            }
        }
    }

    public void updateItem(int itemId, String name, String category, long feeCents, int quantity) throws SQLException {
        String sql = """
                UPDATE items
                SET name = ?, category_id = (SELECT category_id FROM categories WHERE name = ?),
                    rental_fee_cents = ?, quantity = ?
                WHERE item_id = ? AND is_active = TRUE
                  AND ? >= (SELECT COUNT(*) FROM rentals
                            WHERE rentals.item_id = items.item_id AND returned_at IS NULL)
                """;
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement(sql)) {
            statement.setString(1, name);
            statement.setString(2, category);
            statement.setLong(3, feeCents);
            statement.setInt(4, quantity);
            statement.setInt(5, itemId);
            statement.setInt(6, quantity);
            if (statement.executeUpdate() != 1) {
                throw new SQLException("Quantity cannot be lower than the number of active rentals.");
            }
        }
    }

    public void archiveItem(int itemId) throws SQLException {
        String sql = """
                UPDATE items SET is_active = FALSE
                WHERE item_id = ? AND is_active = TRUE
                  AND NOT EXISTS (
                      SELECT 1 FROM rentals
                      WHERE rentals.item_id = items.item_id AND returned_at IS NULL
                  )
                """;
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement(sql)) {
            statement.setInt(1, itemId);
            if (statement.executeUpdate() != 1) {
                throw new SQLException("That item is rented, already archived, or no longer exists.");
            }
        }
    }

    public void rent(String studentId, List<Integer> itemIds, LocalDateTime dueAt)
            throws SQLException {
        String itemSql = "SELECT name, rental_fee_cents, quantity, is_active FROM items WHERE item_id = ? FOR UPDATE";
        try (Connection connection = connect()) {
            connection.setAutoCommit(false);
            try {
                try (PreparedStatement transactionStatement = connection.prepareStatement(
                        "SELECT COALESCE(MAX(transaction_id), 0) + 1 FROM rentals FOR UPDATE");
                        ResultSet transactionResult = transactionStatement.executeQuery()) {
                    transactionResult.next();
                    long transactionId = transactionResult.getLong(1);
                    for (int itemId : itemIds) {
                        long feeCents;
                        int quantity;
                        try (PreparedStatement statement = connection.prepareStatement(itemSql)) {
                            statement.setInt(1, itemId);
                            try (ResultSet result = statement.executeQuery()) {
                                if (!result.next() || !result.getBoolean("is_active")) {
                                    throw new SQLException("One selected item is not available for rental.");
                                }
                                feeCents = result.getLong("rental_fee_cents");
                                quantity = result.getInt("quantity");
                            }
                        }
                        try (PreparedStatement statement = connection.prepareStatement(
                                "SELECT COUNT(*) FROM rentals WHERE item_id = ? AND returned_at IS NULL")) {
                            statement.setInt(1, itemId);
                            try (ResultSet result = statement.executeQuery()) {
                                result.next();
                                if (result.getInt(1) >= quantity) {
                                    throw new SQLException("There are no copies of one selected item left to rent.");
                                }
                            }
                        }
                        try (PreparedStatement statement = connection.prepareStatement("""
                                INSERT INTO rentals
                                    (transaction_id, student_key, item_id, rental_fee_cents, rented_at, due_at)
                                SELECT ?, student_key, ?, ?, ?, ?
                                FROM students WHERE student_id = ?
                                """)) {
                            LocalDateTime rentedAt = LocalDateTime.now();
                            statement.setLong(1, transactionId);
                            statement.setInt(2, itemId);
                            statement.setLong(3, feeCents);
                            statement.setTimestamp(4, Timestamp.valueOf(rentedAt));
                            statement.setTimestamp(5, Timestamp.valueOf(dueAt));
                            statement.setString(6, studentId);
                            if (statement.executeUpdate() != 1) {
                                throw new SQLException("The student profile no longer exists.");
                            }
                        }
                    }
                }
                connection.commit();
            } catch (SQLException exception) {
                connection.rollback();
                throw exception;
            } finally {
                connection.setAutoCommit(true);
            }
        }
    }

    public void returnRental(long rentalId) throws SQLException {
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement(
                "UPDATE rentals SET returned_at = CURRENT_TIMESTAMP WHERE rental_id = ? AND returned_at IS NULL")) {
            statement.setLong(1, rentalId);
            if (statement.executeUpdate() != 1) {
                throw new SQLException("That rental has already been returned or no longer exists.");
            }
        }
    }

    public double totalIncome() throws SQLException {
        try (Connection connection = connect(); Statement statement = connection.createStatement();
                ResultSet result = statement.executeQuery("SELECT COALESCE(SUM(rental_fee_cents), 0) FROM rentals")) {
            result.next();
            return result.getLong(1) / 100.0;
        }
    }

    public boolean hasAdminAccount() throws SQLException {
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement(
                """
                SELECT 1 FROM users u JOIN roles r ON r.role_id = u.role_id
                WHERE r.name = 'ADMIN' LIMIT 1
                """);
                ResultSet result = statement.executeQuery()) {
            return result.next();
        }
    }

    public String getUserPasswordHash(String username) throws SQLException {
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement(
                "SELECT password_hash FROM users WHERE username = ?")) {
            statement.setString(1, username);
            try (ResultSet result = statement.executeQuery()) {
                return result.next() ? result.getString("password_hash") : null;
            }
        }
    }

    public UserAccount getUserAccount(String username) throws SQLException {
        String sql = """
                SELECT u.username, role_lookup.name AS role_name, s.student_id, s.first_name, s.last_name,
                       d.name AS department, s.year_level_id AS year_level
                FROM users u
                JOIN roles role_lookup ON role_lookup.role_id = u.role_id
                LEFT JOIN students s ON s.user_id = u.user_id
                LEFT JOIN departments d ON d.department_id = s.department_id
                WHERE u.username = ?
                """;
        try (Connection connection = connect(); PreparedStatement statement = connection.prepareStatement(sql)) {
            statement.setString(1, username);
            try (ResultSet result = statement.executeQuery()) {
                if (!result.next()) {
                    return null;
                }
                int yearLevelValue = result.getInt("year_level");
                Integer yearLevel = result.wasNull() ? null : yearLevelValue;
                return new UserAccount(result.getString("username"),
                        UserAccount.Role.valueOf(result.getString("role_name")),
                        result.getString("student_id"), result.getString("first_name"),
                        result.getString("last_name"), result.getString("department"),
                        yearLevel);
            }
        }
    }

    public void createStudentAccount(String studentId, String firstName, String lastName, String department,
            int yearLevel, String username, String passwordHash) throws SQLException {
        try (Connection connection = connect()) {
            connection.setAutoCommit(false);
            try {
                long userId = insertStudentUser(connection, username, passwordHash);
                upsertStudentProfile(connection, studentId, firstName, lastName, department, yearLevel, userId);
                connection.commit();
            } catch (SQLException exception) {
                connection.rollback();
                throw exception;
            } finally {
                connection.setAutoCommit(true);
            }
        }
    }

    public void registerStudentAccount(String studentId, String firstName, String lastName, String department,
            int yearLevel, String username, String passwordHash) throws SQLException {
        try (Connection connection = connect()) {
            connection.setAutoCommit(false);
            try {
                long userId = insertStudentUser(connection, username, passwordHash);
                upsertStudentProfile(connection, studentId, firstName, lastName, department, yearLevel, userId);
                connection.commit();
            } catch (SQLException exception) {
                connection.rollback();
                throw exception;
            } finally {
                connection.setAutoCommit(true);
            }
        }
    }

    private long insertStudentUser(Connection connection, String username, String passwordHash) throws SQLException {
        try (PreparedStatement user = connection.prepareStatement("""
                INSERT INTO users (username, password_hash, role_id)
                SELECT ?, ?, role_id FROM roles WHERE name = 'STUDENT'
                """, Statement.RETURN_GENERATED_KEYS)) {
            user.setString(1, username);
            user.setString(2, passwordHash);
            if (user.executeUpdate() != 1) {
                throw new SQLException("The STUDENT role is missing from the roles table.");
            }
            try (ResultSet keys = user.getGeneratedKeys()) {
                if (!keys.next()) {
                    throw new SQLException("The database did not return the new user ID.");
                }
                return keys.getLong(1);
            }
        }
    }

    private void upsertStudentProfile(Connection connection, String studentId, String firstName, String lastName,
            String department, int yearLevel, long userId) throws SQLException {
        try (PreparedStatement student = connection.prepareStatement("""
                INSERT INTO students
                    (student_id, first_name, last_name, department_id, year_level_id, user_id)
                SELECT ?, ?, ?, d.department_id, y.year_level_id, ?
                FROM departments d CROSS JOIN year_levels y
                WHERE d.name = ? AND y.year_level_id = ?
                ON DUPLICATE KEY UPDATE
                    first_name = VALUES(first_name), last_name = VALUES(last_name),
                    department_id = VALUES(department_id), year_level_id = VALUES(year_level_id),
                    user_id = VALUES(user_id)
                """)) {
            student.setString(1, studentId);
            student.setString(2, firstName);
            student.setString(3, lastName);
            student.setLong(4, userId);
            student.setString(5, department);
            student.setInt(6, yearLevel);
            if (student.executeUpdate() == 0) {
                throw new SQLException("The selected department or year level no longer exists.");
            }
        }
    }

    public void createInitialAdmin(String username, String passwordHash) throws SQLException {
        try (Connection connection = connect()) {
            boolean lockAcquired;
            try (PreparedStatement lock = connection.prepareStatement(
                    "SELECT GET_LOCK('school_item_rental_admin_setup', 10)");
                    ResultSet result = lock.executeQuery()) {
                lockAcquired = result.next() && result.getInt(1) == 1;
            }
            if (!lockAcquired) {
                throw new SQLException("Could not acquire the admin setup lock. Try again.");
            }
            try {
                if (hasAdminAccount()) {
                    throw new SQLException("An admin account already exists.");
                }
                try (PreparedStatement insert = connection.prepareStatement("""
                        INSERT INTO users (username, password_hash, role_id)
                        SELECT ?, ?, role_id FROM roles WHERE name = 'ADMIN'
                        """)) {
                    insert.setString(1, username);
                    insert.setString(2, passwordHash);
                    if (insert.executeUpdate() != 1) {
                        throw new SQLException("The ADMIN role is missing from the roles table.");
                    }
                }
            } finally {
                try (PreparedStatement release = connection.prepareStatement(
                        "SELECT RELEASE_LOCK('school_item_rental_admin_setup')")) {
                    release.executeQuery().close();
                }
            }
        }
    }
}
