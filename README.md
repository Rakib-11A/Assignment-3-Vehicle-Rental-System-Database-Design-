
                          Vehicle Rental System Database Design
# Overview

This project implements a comprehensive database design for a vehicle rental system using PostgreSQL. The database schema includes three tables like users, vehicles, and bookings, along with custom data types and triggers(Which I have use for my practice perpose) to maintain data integrity and automate updates. The system supports different user roles (Admin and Customer), various vehicle types (car, bike, truck), and booking statuses to manage the rental process efficiently.

# Database Schema

# Tables

1. users
   - Stores user information including role, name, email, password, and phone
   - Uses an ENUM type for user roles: 'Admin', 'Customer'
   - Includes automatic timestamp tracking with triggers

2. vehicles
   - Contains vehicle details such as name, type, model, registration number, rental price, and availability status
   - Uses ENUM types for vehicle types and availability status
   - Includes automatic timestamp tracking

3. bookings
   - Manages rental bookings with references to users and vehicles
   - Tracks booking dates, status, and total cost
   - Uses an ENUM type for booking statuses: 'pending', 'confirmed', 'cancelled', 'completed'

# Triggers

- Automatic updated_at timestamp updates for all tables using a PostgreSQL trigger function

  CREATE OR REPLACE FUNCTION autometic_updated_at() RETURNS TRIGGER AS $$ 
    BEGIN
      NEW.updated_at = CURRENT_TIMESTAMP;
      RETURN NEW;
    END;
  $$ LANGUAGE plpgsql;


-- Trigger function for users updated_at

  CREATE TRIGGER trigger_users_updated_at
  BEFORE UPDATE ON users 
  FOR EACH ROW 
  EXECUTE FUNCTION autometic_updated_at();

-- Trigger function for vehicles updated_at

  CREATE TRIGGER trigger_vehicle_updated_at
  BEFORE UPDATE ON vehicles
  FOR EACH ROW
  EXECUTE FUNCTION autometic_updated_at();


-- Trigger for bookings updated_at

  CREATE TRIGGER update_bookings_updated_at
  BEFORE UPDATE ON bookings
  FOR EACH ROW 
  EXECUTE FUNCTION autometic_updated_at();

# Custom Types

-- Create ENUM for role

    DROP TYPE IF EXISTS role_type CASCADE;
    CREATE TYPE role_type AS ENUM ('Admin', 'Customer');

-- Create ENUM from vehicle_type

    DROP TYPE IF EXISTS vehicle_type CASCADE;
    CREATE TYPE vehicle_type AS ENUM('car', 'bike', 'truck');

-- Create ENUM for type and status

    DROP TYPE IF EXISTS vehicle_status_type CASCADE;
    CREATE TYPE vehicle_status_type AS ENUM ('available', 'rented', 'maintenance');

-- Create ENUM for status

    DROP TYPE IF EXISTS booking_status_type CASCADE;
    CREATE TYPE booking_status_type AS ENUM ('pending', 'confirmed', 'cancelled', 'completed');
  
# Table Creation

  # users
    CREATE TABLE IF NOT EXISTS users (
      user_id BIGSERIAL PRIMARY KEY,
      role role_type NOT NULL,
      name VARCHAR(100) NOT NULL,
      email VARCHAR(150) UNIQUE NOT NULL,
      password VARCHAR(255) NOT NULL,
      phone VARCHAR(20),

      created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
    );

  #   

# Sample Data

The database includes sample data for demonstration:
- 3 users (1 Admin, 2 Customers)

  INSERT INTO users (name, email, password, phone, role) VALUES
  ('Alice', 'alice@example.com', '$2a$12$R9h/lSAbVLxjCF5nBV5Sbe.S6TfS6S.Gf9/wWwXwYyZz123456789', '1234567890', 'Customer'),
  ('Bob', 'bob@example.com', '$2a$12$KIX9fG3.A.SByM4.S.SByM4.S.SByM4.S.SByM4.S.SByM4.', '0987654321', 'Admin'),
  ('Charlie', 'charlie@example.com', '$2a$12$D4.SByM4.S.SByM4.S.SByM4.S.SByM4.S.SByM4.S.SB', '1122334455', 'Customer');

- 4 vehicles (cars, bikes, trucks with different statuses)

  INSERT INTO vehicles (name, type, model, registration_number, rental_price, status) VALUES
  ('Toyota Corolla', 'car', '2022', 'ABC-123', 50.00, 'available'),
  ('Honda Civic', 'car', '2021', 'DEF-456', 60.00, 'rented'),
  ('Yamaha R15', 'bike', '2023', 'GHI-789', 30.00, 'available'),
  ('Ford F-150', 'truck', '2020', 'JKL-012', 100.00, 'maintenance');

  - 4 bookings with various statuses

    INSERT INTO bookings (user_id, vehicle_id, start_date, end_date, status, total_cost) VALUES
    (1, 2, '2023-10-01', '2023-10-05', 'completed', 240.00),
    (2, 2, '2023-11-01', '2023-11-03', 'completed', 120.00),
    (3, 2, '2023-12-01', '2023-12-02', 'confirmed', 60.00),
    (1, 1, '2023-12-10', '2023-12-12', 'pending', 100.00);

# Queries and Solutions

The following queries demonstrate various SQL operations on the vehicle rental system database. All queries are included in the `Vehicle Rental System Query` file.

# Query 1: Join Operation
Objective: Retrieve booking information along with customer name and vehicle name.

SELECT
  b.booking_id,
  c.name AS customer_name,
  v.name AS vehicle_name,
  b.start_date,
  b.end_date,
  b.booking_status AS status
FROM bookings AS b
INNER JOIN users AS c ON b.user_id = c.user_id
INNER JOIN vehicles AS v ON b.vehicle_id = v.vehicle_id;


Solution: This query uses INNER JOINs to combine data from the bookings, users, and vehicles tables, providing a comprehensive view of each booking with associated customer and vehicle details.

# Query 2: EXISTS Clause
Objective: Find all vehicles that have never been booked.


SELECT
  v.vehicle_id,
  v.name,
  v.type,
  v.model,
  v.registration_number,
  v.rental_price_per_day AS rental_price,
  v.available_status AS status
FROM vehicles AS v
WHERE NOT EXISTS (
  SELECT 1
  FROM bookings AS b
  WHERE b.vehicle_id = v.vehicle_id
)
ORDER BY vehicle_id ASC;

Solution: This query uses the NOT EXISTS clause to identify vehicles that do not have any corresponding records in the bookings table, effectively finding unbooked vehicles.

# Query 3: WHERE Clause with Function
Objective: Retrieve all available vehicles of a specific type (e.g., cars).

Basic Query:

SELECT
    vehicle_id,
    name,
    type,
    model,
    registration_number,
    rental_price_per_day AS rental_price,
    available_status AS status
FROM vehicles
WHERE type = 'car' AND available_status = 'available';


Parameterized Function:

CREATE OR REPLACE FUNCTION get_available_vehicles_by_type(input_type vehicle_type)
RETURNS TABLE (
    vehicle_id BIGINT,
    name VARCHAR,
    type vehicle_type,
    model VARCHAR,
    registration_number VARCHAR,
    rental_price DECIMAL(10, 2),
    status available_status_type
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        v.vehicle_id,
        v.name,
        v.type,
        v.model,
        v.registration_number,
        v.rental_price_per_day,
        v.available_status
    FROM vehicles v
    WHERE v.type = input_type::vehicle_type
    AND v.available_status = 'available';

    IF NOT FOUND THEN
        RAISE NOTICE 'No available vehicles found for type: %', input_type;
    END IF;
END;
$$ LANGUAGE plpgsql;


Usage Examples:

SELECT * FROM get_available_vehicles_by_type('car');
SELECT * FROM get_available_vehicles_by_type('bike');
SELECT * FROM get_available_vehicles_by_type('truck');


Solution: The basic query filters vehicles by type and availability status. The function provides a reusable, parameterized approach to retrieve available vehicles of any specified type, with error handling for cases where no vehicles are found.

# Query 4: GROUP BY and HAVING
Objective: Find the total number of bookings for each vehicle and display only those vehicles that have more than 2 bookings.


SELECT
    v.name AS vehicle_name,
    COUNT(b.booking_id) AS total_bookings
FROM vehicles v
INNER JOIN bookings b ON v.vehicle_id = b.vehicle_id
GROUP BY v.name
HAVING COUNT(b.booking_id) > 2
ORDER BY total_bookings DESC;


Solution: This query groups bookings by vehicle name, counts the total bookings per vehicle, and filters the results to show only vehicles with more than 2 bookings, ordered by the total number of bookings in descending order.

# Setup Instructions

1. Ensure PostgreSQL is installed and running on your system.
2. Create a new database for the vehicle rental system.
3. Execute the SQL commands in the `Vehicle Rental System Query` file in order.
4. The file includes schema creation, data insertion, and query examples.

# Notes

- Indexes for the bookings table are could be user for large datasets.
- All numeric values use DECIMAL for precision.
- Timestamps use TIMESTAMPTZ for timezone awareness.
- The system includes data validation constraints (e.g., non-negative rental prices and total costs).