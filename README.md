# BookStoreApp — Spring Boot Book Exchange Platform

Web εφαρμογή ανταλλαγής βιβλίων. Χτισμένη με Java Spring Boot, Spring Security, JPA/Hibernate και Thymeleaf templates. University project.

## Tech Stack

- **Java** με **Spring Boot**
- **Spring Security** — BCrypt + custom success handler
- **Spring Data JPA** + **Hibernate** — ORM για MySQL
- **Thymeleaf** — server-side templates
- **MySQL** — βάση δεδομένων
- **Bootstrap 4** — frontend styling

## Αρχιτεκτονική

```
src/main/java/com/example/BookStoreApp/
├── config/
│   ├── WebSecurityConfig.java          # Security rules + auth provider
│   ├── CustomSecuritySuccessHandler.java # Role-based redirect μετά το login
│   └── WebMvcConfig.java               # "/" → homepage view
├── controller/
│   ├── AuthController.java             # /login, /register, /save
│   ├── UserController.java             # Όλες οι user actions
│   └── AdminController.java            # /admin/dashboard
├── model/
│   ├── User.java                       # Spring Security UserDetails
│   ├── UserProfile.java                # Extended profile (address, age, κ.α.)
│   ├── Book.java                       # Βιβλίο offer
│   ├── BookAuthor.java
│   ├── BookCategory.java
│   └── Notify.java                     # Notifications
├── dao/                                # JPA Repositories
├── service/                            # Business logic layer
├── formsdata/                          # DTO objects για forms
└── strategies/                         # Search Strategy Pattern
```

## Database Schema (JPA entities)

### User — UserProfile (1:1 via username)
```
users        → id, user_name (unique), password, role (USER|ADMIN)
profiles     → user_name (PK), address, age, phoneNumber
```

### Books & Relationships
```
books              → book_id, title, summary, userProfile_id (FK), category_id (FK)
authors            → author_id, name
categories         → category_id, name

book_wrote         → book_id, author_id          (Book ↔ BookAuthor, M:N)
requested_books    → book_id, user_name           (Book ↔ UserProfile, M:N)
user_favourite_authors    → user_name, author_id  (UserProfile ↔ BookAuthor, M:N)
user_favourite_categories → user_name, category_id
```

### Notifications
```
notify → notify_id, userProfile_id (FK), notification_message
```

## Authentication & Authorization

**Roles**: `USER`, `ADMIN`

| Path | Access |
|------|--------|
| `/`, `/login`, `/register`, `/save`, `/makeprofile`, `/saveprofile` | Public |
| `/user/**` | `USER` role |
| `/admin/**` | `ADMIN` role |

**Login flow:**
1. `POST /login` → Spring Security form login
2. `CustomSecuritySuccessHandler` redirects:
   - `ADMIN` → `/admin/dashboard`
   - `USER` → `/user/dashboard`

**Password**: BCrypt encoding μέσω `BCryptPasswordEncoder`

## Registration Flow

```
POST /register → GET /register (form)
POST /save     → Δημιουργία User (BCrypt password) → redirect σε create profile
POST /makeprofile → Εμφάνιση profile form
POST /saveprofile → Αποθήκευση UserProfile (address, age, phone, favourite authors/categories)
```

## Features

### Book Offers
- Κάθε user μπορεί να προσθέσει βιβλία που θέλει να ανταλλάξει
- Κάθε βιβλίο έχει: τίτλο, summary, authors (comma-separated), κατηγορία
- Authors: αν δεν υπάρχουν → δημιουργούνται αυτόματα

### Search (Strategy Pattern)
Δύο στρατηγικές αναζήτησης:

| Strategy | Περιγραφή |
|----------|-----------|
| `ExactSearchStrategy` | Ακριβής τίτλος + ακριβώς οι ίδιοι authors |
| `ApproximateSearchStrategy` | `LIKE` για τίτλο + οι επιλεγμένοι authors να υπάρχουν (subset) |

Και οι δύο κληρονομούν από `TemplateSearchStrategy` που υλοποιεί το template method pattern:
```
search() {
  makeInitialListOfBooks()  // abstract — ορίζεται υποκλάση
  checkIfAuthorsMatch()     // abstract
  filterOutCurrentUser()    // κοινό
  mapToFormData()           // κοινό
}
```

### Book Requests
- User A βλέπει βιβλία άλλων users (φιλτράρει τα δικά του)
- Πατά "Request" → προστίθεται στη `requestingUsers` list του βιβλίου
- Owner βλέπει requests → πατά "Accept"
- Accept → `Notify` εγγραφές για όλους τους requesters (success/failure message)
- Βιβλίο διαγράφεται μετά το accept

### Notifications
- Ordered by `notifyId DESC` (νεότερα πρώτα)
- Εμφανίζεται ως λίστα string μηνυμάτων

## Book Categories (Pre-seeded)

Art, Comic, Fantasy, Fiction, Biographies, History, Science, Literature, Adventure, Crime, Other

Seed γίνεται στο startup αν ο πίνακας είναι κενός (`CommandLineRunner`).

## Configuration

```properties
# application.properties
spring.datasource.url=jdbc:mysql://localhost:3306/bookstoreapp?user=root
spring.datasource.username=root
spring.datasource.password=YOUR_PASSWORD
spring.jpa.hibernate.ddl-auto=update
```

## Run Locally

```bash
# Δημιουργία database
mysql -u root -p
CREATE DATABASE bookstoreapp;

# Run
mvn spring-boot:run
# → http://localhost:8080
```

## Pages (Thymeleaf Templates)

| Template | URL | Περιγραφή |
|----------|-----|-----------|
| `homepage.html` | `/` | Landing με login/register links |
| `auth/signin.html` | `/login` | Login form |
| `auth/signup.html` | `/register` | Registration form |
| `user/dashboard.html` | `/user/dashboard` | User home |
| `admin/dashboard.html` | `/admin/dashboard` | Admin home |
| `addbook/bookform.html` | `/bookform` | Προσθήκη βιβλίου |
| `bookoffers/booklist.html` | `/booklist` | Τα βιβλία μου |
| `search/searchForm.html` | `/searchForm` | Αναζήτηση |
| `search/booklist.html` | `/search` (POST) | Αποτελέσματα |
| `requests/requests.html` | `/showRequestingUsersForBookOffer` | Αιτήσεις |
| `notify/notify.html` | `/showNotifications` | Ειδοποιήσεις |
| `userProfile/userProfile.html` | `/makeprofile` | Δημιουργία profile |
