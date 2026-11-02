# SCHOOL_ITEM_RENTAL_SYSTEM
# -- phpMyAdmin SQL Dump -- version 5.2.1 -- https://www.phpmyadmin.net/
-- Host: 127.0.0.1
-- Generation Time: Oct 09, 2026 at 08:53 PM
-- Server version: 10.4.32-MariaDB
-- PHP Version: 8.2.12

SET SQL_MODE = "NO_AUTO_VALUE_ON_ZERO";
START TRANSACTION;
SET time_zone = "+00:00";


/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;

--
# -- Database: `school_item_rental`
--

-- --------------------------------------------------------

--
# -- Table structure for table `categories`
--

CREATE TABLE `categories` (
  `category_id` int(10) UNSIGNED NOT NULL,
  `name` varchar(100) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `categories`
--

INSERT INTO `categories` (`category_id`, `name`) VALUES
(3, 'Electronics'),
(4, 'Laboratory Equipment'),
(5, 'Other'),
(2, 'Personal Items'),
(1, 'School Supplies');

-- --------------------------------------------------------

--
# -- Table structure for table `departments`
--

CREATE TABLE `departments` (
  `department_id` int(10) UNSIGNED NOT NULL,
  `name` varchar(100) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `departments`
--

INSERT INTO `departments` (`department_id`, `name`) VALUES
(8, 'BEED'),
(4, 'BSA'),
(3, 'BSBA'),
(6, 'BSCRIM'),
(2, 'BSCS'),
(7, 'BSED'),
(5, 'BSHM'),
(1, 'BSIT'),
(9, 'CITE');

-- --------------------------------------------------------

--
# -- Table structure for table `items`
--

CREATE TABLE `items` (
  `item_id` int(10) UNSIGNED NOT NULL,
  `name` varchar(120) NOT NULL,
  `category_id` int(10) UNSIGNED NOT NULL,
  `rental_fee_cents` bigint(20) UNSIGNED NOT NULL,
  `quantity` int(10) UNSIGNED NOT NULL DEFAULT 1,
  `is_active` tinyint(1) NOT NULL DEFAULT 1
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `items`
--

INSERT INTO `items` (`item_id`, `name`, `category_id`, `rental_fee_cents`, `quantity`, `is_active`) VALUES
(1, 'Scientific Calculator', 1, 5000, 3, 3),
(2, 'Umbrella', 2, 1500, 3, 3),
(3, 'Charger', 3, 2500, 3, 3),
(4, 'Lab Equipment', 4, 5000, 3, 3),
(5, 'HDMI', 3, 5000, 3, 3),
(6, 'Laptop', 3, 5000, 3, 3),
(7, 'Power Banks', 3, 3000, 3, 3);

-- --------------------------------------------------------

--
# -- Table structure for table `rentals`
--

CREATE TABLE `rentals` (
  `rental_id` int(10) UNSIGNED NOT NULL,
  `transaction_id` int(10) UNSIGNED NOT NULL,
  `student_key` int(10) UNSIGNED NOT NULL,
  `item_id` int(10) UNSIGNED NOT NULL,
  `rental_fee_cents` bigint(20) UNSIGNED NOT NULL,
  `rented_at` datetime NOT NULL DEFAULT current_timestamp(),
  `due_at` datetime NOT NULL,
  `returned_at` datetime DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `rentals`
--

INSERT INTO `rentals` (`rental_id`, `transaction_id`, `student_key`, `item_id`, `rental_fee_cents`, `rented_at`, `due_at`, `returned_at`) VALUES
(1, 1, 2, 4, 5000, '2026-10-03 16:57:50', '2026-10-11 15:59:59', NULL),
(2, 2, 4, 1, 2000, '2026-10-07 06:20:22', '2026-10-14 15:59:59', '2026-10-07 14:26:05'),
(3, 3, 1, 2, 1500, '2026-10-09 12:02:17', '2026-10-16 15:59:59', '2026-10-09 20:02:41');

-- --------------------------------------------------------

--
# -- Table structure for table `roles`
--

CREATE TABLE `roles` (
  `role_id` int(10) UNSIGNED NOT NULL,
  `name` varchar(20) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `roles`
--

INSERT INTO `roles` (`role_id`, `name`) VALUES
(1, 'ADMIN'),
(2, 'STUDENT');

-- --------------------------------------------------------

--
# -- Table structure for table `students`
--

CREATE TABLE `students` (
  `student_key` int(10) UNSIGNED NOT NULL,
  `student_id` varchar(30) NOT NULL,
  `first_name` varchar(120) NOT NULL,
  `last_name` varchar(120) NOT NULL,
  `department_id` int(10) UNSIGNED DEFAULT NULL,
  `year_level_id` int(10) UNSIGNED DEFAULT NULL,
  `user_id` int(10) UNSIGNED DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `students`
--

INSERT INTO `students` (`student_key`, `student_id`, `first_name`, `last_name`, `department_id`, `year_level_id`, `user_id`) VALUES
(1, '02-2526-013803', 'Wenz', 'Borja', 9, 2, 5),
(2, '123456789', 'Arjay', 'Siberia', 1, 3, 2),
(3, '1234567890', 'Mcmiel', 'Ganora', 9, 3, 4),
(4, '2526031407', 'Reniel', 'Montes', 9, 2, 3);

-- --------------------------------------------------------

--
# -- Table structure for table `users`
--

CREATE TABLE `users` (
  `user_id` int(10) UNSIGNED NOT NULL,
  `username` varchar(80) NOT NULL,
  `password_hash` varchar(255) NOT NULL,
  `role_id` int(10) UNSIGNED NOT NULL,
  `created_at` datetime NOT NULL DEFAULT current_timestamp()
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `users`
--

INSERT INTO `users` (`user_id`, `username`, `password_hash`, `role_id`, `created_at`) VALUES
(1, 'admin', 'pbkdf2-sha256$210000$Cnn+/yE/vQqdxzK/MDluQw==$I/ybyJg65Zv4PsujsZktEs+ZsSDmmt99L9+/M8PteMc=', 1, '2026-09-29 13:25:31'),
(2, 'Arjaypax', 'pbkdf2-sha256$210000$xbC9y7q7wtN1IzKcHTnmDA==$kRI2bfnL9AeAd1WujmZ13yeAWGwenTyRqdXF+IMGxL8=', 2, '2026-10-04 00:57:31'),
(3, 'Reynel', 'pbkdf2-sha256$210000$JqAK3mQi1pNg1F4u2he8Cg==$rujB0qIQbJmCPlCTSBqScrrJc5YEDYrIIwhOeB9peSE=', 2, '2026-10-07 14:19:43'),
(4, 'Pasikat', 'pbkdf2-sha256$210000$lIZ/2hiVf0Bi9sreNKu70Q==$rSOdxZ6EVBiB5KSoj1rqpwGK80dzr7LBdQTplNg5VJ4=', 2, '2026-10-09 16:23:28'),
(5, 'wenz', 'pbkdf2-sha256$210000$4FjwU8nsLtcPZnAm0IABCA==$AoVbCTjSTwK51MsrdK9u277PQSYvaUaDliYO12L1MqU=', 2, '2026-10-09 20:01:14');

-- --------------------------------------------------------

--
# -- Table structure for table `year_levels`
--

CREATE TABLE `year_levels` (
  `year_level_id` int(10) UNSIGNED NOT NULL,
  `name` varchar(40) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

--
# -- Dumping data for table `year_levels`
--

INSERT INTO `year_levels` (`year_level_id`, `name`) VALUES
(1, 'First Year'),
(4, 'Fourth Year'),
(2, 'Second Year'),
(3, 'Third Year');

--
# -- Indexes for dumped tables

-- Indexes for table `categories`
--
ALTER TABLE `categories`
  ADD PRIMARY KEY (`category_id`),
  ADD UNIQUE KEY `uq_categories_name` (`name`);

--
# -- Indexes for table `departments`
--
ALTER TABLE `departments`
  ADD PRIMARY KEY (`department_id`),
  ADD UNIQUE KEY `uq_departments_name` (`name`);

--
# -- Indexes for table `items`
--
ALTER TABLE `items`
  ADD PRIMARY KEY (`item_id`),
  ADD UNIQUE KEY `uq_items_name` (`name`),
  ADD KEY `fk_items_category` (`category_id`);

--
# -- Indexes for table `rentals`
--
ALTER TABLE `rentals`
  ADD PRIMARY KEY (`rental_id`),
  ADD KEY `rentals_by_transaction` (`transaction_id`),
  ADD KEY `rentals_by_student` (`student_key`),
  ADD KEY `rentals_by_due_date` (`due_at`),
  ADD KEY `rentals_by_item` (`item_id`);

--
# -- Indexes for table `roles`
--
ALTER TABLE `roles`
  ADD PRIMARY KEY (`role_id`),
  ADD UNIQUE KEY `uq_roles_name` (`name`);

--
# -- Indexes for table `students`
--
ALTER TABLE `students`
  ADD PRIMARY KEY (`student_key`),
  ADD UNIQUE KEY `uq_students_school_id` (`student_id`),
  ADD UNIQUE KEY `uq_students_user_id` (`user_id`),
  ADD KEY `fk_students_department` (`department_id`),
  ADD KEY `fk_students_year_level` (`year_level_id`);

--
# -- Indexes for table `users`
--
ALTER TABLE `users`
  ADD PRIMARY KEY (`user_id`),
  ADD UNIQUE KEY `username` (`username`),
  ADD KEY `fk_users_role` (`role_id`);

--
# -- Indexes for table `year_levels`
--
ALTER TABLE `year_levels`
  ADD PRIMARY KEY (`year_level_id`),
  ADD UNIQUE KEY `uq_year_levels_name` (`name`);

--
# -- AUTO_INCREMENT for dumped tables

# -- AUTO_INCREMENT for table `categories`
--
ALTER TABLE `categories`
  MODIFY `category_id` int(10) UNSIGNED NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=6;

--
# -- AUTO_INCREMENT for table `departments`
--
ALTER TABLE `departments`
  MODIFY `department_id` int(10) UNSIGNED NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=10;

--
-- AUTO_INCREMENT for table `items`
--
ALTER TABLE `items`
  MODIFY `item_id` int(10) UNSIGNED NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=8;

--
# -- AUTO_INCREMENT for table `rentals`
--
ALTER TABLE `rentals`
  MODIFY `rental_id` int(10) UNSIGNED NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=4;

--
# -- AUTO_INCREMENT for table `students`
--
ALTER TABLE `students`
  MODIFY `student_key` int(10) UNSIGNED NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=5;

--
# -- AUTO_INCREMENT for table `users`
--
ALTER TABLE `users`
  MODIFY `user_id` int(10) UNSIGNED NOT NULL AUTO_INCREMENT, AUTO_INCREMENT=6;

--
# -- Constraints for dumped tables

# -- Constraints for table `items`
--
ALTER TABLE `items`
  ADD CONSTRAINT `fk_items_category` FOREIGN KEY (`category_id`) REFERENCES `categories` (`category_id`);

--
# -- Constraints for table `rentals`
--
ALTER TABLE `rentals`
  ADD CONSTRAINT `fk_rentals_item` FOREIGN KEY (`item_id`) REFERENCES `items` (`item_id`),
  ADD CONSTRAINT `fk_rentals_student` FOREIGN KEY (`student_key`) REFERENCES `students` (`student_key`);

--
# -- Constraints for table `students`
--
ALTER TABLE `students`
  ADD CONSTRAINT `fk_students_department` FOREIGN KEY (`department_id`) REFERENCES `departments` (`department_id`),
  ADD CONSTRAINT `fk_students_user` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`) ON DELETE SET NULL,
  ADD CONSTRAINT `fk_students_year_level` FOREIGN KEY (`year_level_id`) REFERENCES `year_levels` (`year_level_id`);

--
# -- Constraints for table `users`
--
ALTER TABLE `users`
  ADD CONSTRAINT `fk_users_role` FOREIGN KEY (`role_id`) REFERENCES `roles` (`role_id`);
COMMIT;

/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
