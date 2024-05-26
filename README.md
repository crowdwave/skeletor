Here is the updated documentation with separate sections for users who already have a users table and those who do not:

# Identoro User Signup Server for SQLite or PostgreSQL

This is a Go web server that supports user signup, signin, password reset, and account verification functionalities. It uses PostgreSQL (via `pgx` driver) or SQLite for data storage and includes rate limiting for login attempts per account.

## Features

- User Signup
- User Signin
- Password Reset
- Account Verification
- CSRF Protection
- Rate Limiting (15 login attempts per account per hour)
- Configurable via Environment Variables
- Graceful Shutdown on SIGINT or SIGTERM

## Installation

1. **Clone the repository:**

    ```bash
    git clone https://github.com/your-repo/go-web-server.git
    cd go-web-server
    ```

2. **Install dependencies:**

    ```bash
    go mod tidy
    ```

3. **Create a `.env` file:**

    ```bash
    cp .env.example .env
    ```

4. **Update the `.env` file with your configuration:**

    ```
    DB_TYPE=postgres
    DATABASE_URL=your_database_url
    SECRET_KEY=your_secret_key
    EMAIL_SENDER=your_email_sender
    EMAIL_PASSWORD=your_email_password
    SMTP_HOST=your_smtp_host
    SMTP_PORT=your_smtp_port
    WEB_SERVER_ADDRESS=http://localhost:8080
    EMAIL_REPLY_TO=your_reply_to_email
    RECAPTCHA_SITE_KEY=your_recaptcha_site_key
    RECAPTCHA_SECRET_KEY=your_recaptcha_secret_key
    ```

## Usage

1. **Run the server:**

    ```bash
    go run main.go
    ```

2. **Access the server:**

    Open your browser and go to `http://localhost:8080`

## API Endpoints

- `/signup` - User signup
- `/signin` - User signin
- `/signout` - User signout
- `/forgot` - Password reset request
- `/reset` - Password reset
- `/verify` - Account verification

## Configuration

The server can be configured using the following environment variables:

- `DB_TYPE` - Type of database (`postgres` or `sqlite`)
- `DATABASE_URL` - Connection string for the database
- `SECRET_KEY` - Secret key for session cookies and CSRF protection
- `EMAIL_SENDER` - Email address used for sending emails
- `EMAIL_PASSWORD` - Password for the email sender
- `SMTP_HOST` - SMTP server host
- `SMTP_PORT` - SMTP server port
- `WEB_SERVER_ADDRESS` - Address of the web server
- `EMAIL_REPLY_TO` - Reply-to email address
- `RECAPTCHA_SITE_KEY` - Site key for Google reCAPTCHA
- `RECAPTCHA_SECRET_KEY` - Secret key for Google reCAPTCHA

## Rate Limiting

The server implements rate limiting to prevent abuse of the login functionality. Each account (username) is limited to 15 login attempts per hour. If the limit is exceeded, a "Too Many Requests" response is returned.

## Graceful Shutdown

The server handles `SIGINT` and `SIGTERM` signals for graceful shutdown, ensuring that the database connection is properly closed before the server exits.

## Integration with PostgreSQL 

### If you already have a users table

Identoro works with your existing Postgres users table via Postgres views which map our SQL queries to your table and field names. You need to create the necessary views on your Postgres server, ensuring the required fields are present and have matching column data types.

The view names must be exactly as shown below and all fields must be present - your job is to put in the name of your users table and the names of the fields in your users table.

### Example:

Assuming your actual `users` table (`actual_users_table`) has the following schema:

```sql
CREATE TABLE actual_users_table (
    user_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_name VARCHAR(50) NOT NULL,
    passwd VARCHAR(100) NOT NULL,
    mail VARCHAR(100) NOT NULL,
    reset_token VARCHAR(50),
    is_verified BOOLEAN NOT NULL DEFAULT FALSE,
    verification_token VARCHAR(50)
);
```

Your view should ensure that the columns have the same data types:

```sql
CREATE VIEW identoro_users AS
SELECT 
    user_id, 
    user_name AS username, 
    passwd AS password, 
    mail AS email, 
    reset_token, 
    is_verified AS verified, 
    verification_token 
FROM actual_users_table;

CREATE RULE insert_identoro_users AS
ON INSERT TO identoro_users
DO INSTEAD
INSERT INTO actual_users_table (user_name, passwd, mail, verification_token) 
VALUES (NEW.username, NEW.password, NEW.email, NEW.verification_token);

CREATE RULE update_identoro_users AS
ON UPDATE TO identoro_users
DO INSTEAD
UPDATE actual_users_table
SET 
    user_name = NEW.username,
    passwd = NEW.password,
    mail = NEW.email,
    reset_token = NEW.reset_token,
    is_verified = NEW.verified,
    verification_token = NEW.verification_token
WHERE user_id = NEW.user_id;

CREATE RULE delete_identoro_users AS
ON DELETE TO identoro_users
DO INSTEAD
DELETE FROM actual_users_table
WHERE user_id = OLD.user_id;
```

### Key Points to Ensure

- **Matching Data Types**: Ensure that the data types of the columns in the view match the data types of the columns in the underlying table. This is critical for insert, update, and select operations to work correctly.
- **Constraints and Defaults**: Any constraints (like `NOT NULL`, `DEFAULT`, etc.) or default values should also be considered to ensure data integrity.
- **Indexes**: Indexes on the underlying table can help optimize the performance of queries on the view.

### If you do not already have a users table

If you do not have an existing users table, you can use the following SQL to create both the needed table and the views:

```sql
-- Create the users table
CREATE TABLE actual_users_table (
    user_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_name VARCHAR(50) NOT NULL,
    passwd VARCHAR(100) NOT NULL,
    mail VARCHAR(100) NOT NULL,
    reset_token VARCHAR(50),
    is_verified BOOLEAN NOT NULL DEFAULT FALSE,
    verification_token VARCHAR(50)
);

-- Create the identoro_users view
CREATE VIEW identoro_users AS
SELECT 
    user_id, 
    user_name AS username, 
    passwd AS password, 
    mail AS email, 
    reset_token, 
    is_verified AS verified, 
    verification_token 
FROM actual_users_table;

-- Create insert rule for the view
CREATE RULE insert_identoro_users AS
ON INSERT TO identoro_users
DO INSTEAD
INSERT INTO actual_users_table (user_name, passwd, mail, verification_token) 
VALUES (NEW.username, NEW.password, NEW.email, NEW.verification_token);

-- Create update rule for the view
CREATE RULE update_identoro_users AS
ON UPDATE TO identoro_users
DO INSTEAD
UPDATE actual_users_table
SET 
    user_name = NEW.username,
    passwd = NEW.password,
    mail = NEW.email,
    reset_token = NEW.reset_token,
    is_verified = NEW.verified,
    verification_token = NEW.verification_token
WHERE user_id = NEW.user_id;

-- Create delete rule for the view
CREATE RULE delete_identoro_users AS
ON DELETE TO identoro_users
DO INSTEAD
DELETE FROM actual_users_table
WHERE user_id = OLD.user_id;
```

By ensuring data type consistency and respecting constraints, your views will correctly map the columns and provide seamless integration with the Identoro server's queries.

## License

This project is licensed under the MIT License.
