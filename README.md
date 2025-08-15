# CineBook Application

This is the central manifest for CineBook Application, a distributed system built with microservices.

## Microservices Architecture

Here are the primary services that make up the application. See each repository for specific details.

* **[auth-service](https://github.com/KrishnaPaliwal/auth-service)**: Manages user accounts, authentication, and profiles.
* **[cinema-service](https://github.com/KrishnaPaliwal/cinema-service)**: Manages movies and its related functionality.
* **[booking-service](https://github.com/KrishnaPaliwal/booking-service)**: Manages business functionality of booking flow.
* **[notification-service](https://github.com/KrishnaPaliwal/notification-service)**: Sends emails, messages and push notifications.
* **[api-gateway](https://github.com/KrishnaPaliwal/api-gateway)**: The single entry point for all client requests.
* **[cinebook-infra](https://github.com/KrishnaPaliwal/cinebook-infra)**: Configuration files for CineBook Application.
* **[cinebook-frontend](https://github.com/KrishnaPaliwal/cinebook-frontend)**: Frontend GUI project.

## High-Level Architecture
Architecture: This is a well-defined microservice architecture. Each service has a clear responsibility, and they communicate effectively through REST APIs (for synchronous calls) and RabbitMQ (for asynchronous notifications).

Authentication & Authorization: The auth-service handles user registration (with OTP) and login, issuing JWTs. The other services (cinema-service in particular) correctly use a JWT filter to validate these tokens and enforce role-based access (ROLE_ADMIN vs. ROLE_USER).

Booking Flow: The booking process is robust, following a "lock-then-pay" model. It correctly interacts with the cinema-service for show details and the payment-service for transactions.

Notifications: The notification-service is properly decoupled and handles both email and SMS notifications for OTP and booking confirmations.

Frontend: The React application uses a modern stack with Vite, Material UI, and a component-based structure. Global state for authentication is managed well with a Context.

## Functional Features
Core User & Authentication Features
User Registration with OTP: New users can sign up with their name, email, password, and mobile number. The system sends a unique One-Time Password (OTP) via both email and SMS (using Twilio) to verify their contact information before the account is activated.

User Login: Registered and verified users can log in securely. The system uses JWT (JSON Web Tokens) to manage sessions, ensuring that subsequent requests are authenticated.

Forgot Password: Users who have forgotten their password can request a reset link to be sent to their registered email address. The link contains a secure, single-use token.

Password Reset: Users can follow the link from the email to a dedicated page where they can set a new password for their account.

User Profile Management: Logged-in users can view their profile information (name, email, mobile number) and have the functionality to change their password securely by providing their current password.

Core Booking & Payment Flow
Movie & Showtime Browsing: Users can view a list of all movies currently showing. They can click on a movie to see all available showtimes.

Seat Selection: For a specific showtime, users are presented with a visual seat map where they can select available seats. Booked seats are clearly marked as unavailable.

Seat Locking: To prevent race conditions, selected seats are temporarily locked for a short duration via a backend process, ensuring no other user can book the same seats while the current user proceeds to payment.

Payment Integration (Razorpay): The application is fully integrated with the Razorpay payment gateway. After selecting seats, users are redirected to a secure payment page to complete the transaction using various methods like UPI, cards, etc.

Booking Confirmation & Notifications: Upon successful payment (confirmed via a webhook from Razorpay), the booking status is updated in the database, and the user receives a confirmation via both email and SMS.

Admin & Content Management
Role-Based Access Control: The application distinguishes between regular users (ROLE_USER) and administrators (ROLE_ADMIN). Admin-only features are protected.

Admin Dashboard: A dedicated section for administrators to manage the application's content.

Add New Movie: Admins can add new movies to the system, including details like title, genre, and an image URL.

Add New Cinema & Screens: Admins can add new cinema locations, including the cinema's name, address, city, and details about each screen within that cinema (e.g., screen number, total seats).

Add New Showtime: Admins can create new showtimes for any movie at any available screen, setting the specific date, time, and price per seat.

UI/UX & Personalization Features
Location Setting:

Dynamic City List: The list of available cities in the navigation bar is dynamically populated from the cinemas listed in the database.

Geolocation: Users can click an icon to use their browser's location services to automatically detect their city and update the movie listings.

Location-Based Filtering: The homepage displays only the movies that have showtimes in the user's selected city.

Booking History:

Tabbed View: Logged-in users can view their bookings, neatly separated into "Upcoming" and "Past" tabs.

Detailed Booking Cards: Each booking is displayed with the movie poster, title, showtime, seat numbers, and status (e.g., CONFIRMED, CANCELLED).

PDF Ticket Download: For confirmed upcoming bookings, users can download a full e-ticket in PDF format, which includes a scannable QR code and all booking details.

Booking Cancellation & Refund:

Users can cancel their upcoming, confirmed bookings directly from the Booking History page.

This action automatically triggers a full refund process via the Razorpay API and makes the seats available again for other users.

## How to Run
Deploy these services on Google Cloud GKE Cluster.
