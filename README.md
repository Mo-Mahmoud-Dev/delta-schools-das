# Delta American Schools

Delta American Schools (DAS) is a bilingual website for the school in Mansoura, Egypt. It combines a static, responsive school website with an Express server that provides news and course management, student accounts, application and visit email forms, and an AI chat endpoint.

## Contents

- [Features](#features)
- [Technology](#technology)
- [Project structure](#project-structure)
- [Run locally](#run-locally)
- [Configuration](#configuration)
- [Pages and routes](#pages-and-routes)
- [API overview](#api-overview)
- [Data and uploads](#data-and-uploads)
- [Deployment](#deployment)
- [Development notes](#development-notes)

## Features

- **School information:** Home, About Us, Services, Admission, and Gallery pages cover the school's programs, academic stages, staff, facilities, contact details, and history.
- **English and Arabic:** `locales/en.json` and `locales/ar.json` supply translations. The shared `i18n.js` helper applies translations, switches page direction for Arabic, and saves the selected language in browser local storage.
- **News:** Visitors can browse posts by category and read them in a detail modal. Administrators can create bilingual posts with an optional image and delete existing posts.
- **Student courses:** Students can register and sign in to view courses by grade, with optional YouTube lessons and linked PDF materials. Instructors can manage courses and student accounts.
- **Admission applications:** Applicants can submit their contact details and a PDF, ZIP, or RAR document. The server sends the submission and attachment by email.
- **Campus visit requests:** Visitors can request a date and provide notes; the server emails the request to the school's configured recipient.
- **AI assistant:** A dedicated chat page sends conversations to the server, which calls the Groq chat-completions API.
- **Responsive presentation:** The site uses page-specific stylesheets, shared image assets, responsive navigation, and third-party fonts and UI libraries.

## Technology

- Front end: HTML, CSS, SCSS, and browser JavaScript; there is no front-end framework or bundler configured.
- Server: Node.js and Express 4.
- Persistence: MongoDB accessed using Mongoose. Student passwords are hashed with `bcryptjs`.
- Email: Resend.
- File handling: Multer.
- Hosting configuration: Railway with the Nixpacks builder.
- External browser resources include Google Fonts, Font Awesome, Swiper, Leaflet, Fancybox, and YouTube thumbnails/embeds.

Use **Node.js 20 or later**. The installed Resend dependency requires Node.js 20+.

## Project structure

| Path | Purpose |
| --- | --- |
| `index.html`, `script.js`, `style.css`, `style.scss` | Home page, shared navigation behavior, and global styles. |
| `aboutUs/` | School history, founders, staff, and timeline page. |
| `services/` | School services information page. |
| `admission/` | Academic stages, fees, requirements, and admissions information. |
| `sendAdmissionPapers/pages/` | Admission application form and its styles. |
| `SendTheVisitAppointment/` | Campus visit request form and its styles. |
| `courses/` | Student login/registration, course listing, and instructor management pages. |
| `News/` | Public news page, administrator page, and local news image upload directory. |
| `Gallery/` | School photo gallery and gallery styles/scripts. |
| `my-chatbot/public/` | Standalone AI chat page. |
| `locales/` | English and Arabic translation dictionaries. |
| `pictures/` | School logos, staff/founder portraits, gallery photographs, and illustration assets. |
| `Server/server.js` | Express app, page routes, API endpoints, database models, uploads, and external-service integrations. |
| `Server/package.json` | Server dependencies and server-specific start/development scripts. |
| `railway.json` | Railway build and start configuration. |

The `.css` files are the stylesheets served by the pages. Matching `.scss` files are also present for several page styles; neither package manifest currently defines an SCSS compilation script.

## Run locally

1. Install Node.js 20+ and make a MongoDB instance available.
2. From the repository root, install dependencies:

   ```sh
   npm install
   ```

   The root package's `postinstall` script installs the server dependencies in `Server/`.
3. Create `Server/.env` and configure the values described in [Configuration](#configuration).
4. Start the app from the repository root:

   ```sh
   npm start
   ```

   The server listens on `http://localhost:3000` by default and serves both the pages and API. Open `http://localhost:3000/` to view the home page.

For server development with automatic restarts:

```sh
cd Server
npm run dev
```

If using the development command, install the server dependencies first if they are not already present:

```sh
npm install
```

The application expects MongoDB to be available at startup. Without a configured MongoDB URL it attempts to connect to `mongodb://localhost:27017/dasdb`. A Resend API key is required for the application and visit email forms; the chatbot also depends on valid Groq API access.

## Configuration

The server loads environment variables with `dotenv`. For local development, put them in `Server/.env`; do not commit credentials.

| Variable | Purpose | Default/behavior |
| --- | --- | --- |
| `PORT` | HTTP port used by Express. | `3000` |
| `MONGODB_URI` | MongoDB connection string. | Takes precedence over the alternatives below. |
| `MONGO_URL` | Alternate MongoDB connection string, including Railway's MongoDB plugin variable. | Used if `MONGODB_URI` is unset. |
| `MONGODB_URL` | Additional MongoDB connection-string alternative. | Used if the two variables above are unset. |
| `RESEND_API_KEY` | Resend credential used to send application and visit emails. | Required for email delivery. |
| `ADMIN_PASSWORD` | Password for the News administration API. | The source contains a development fallback; set a unique value for deployment. |
| `INSTRUCTOR_PASSWORD` | Password for the courses instructor panel. | The source contains a development fallback; set a unique value for deployment. |
| `ALLOWED_ORIGIN` | CORS allowed origin. | `*` if unset. |
| `UPLOADS_DIR` | Directory for uploaded news images. | `News/uploads/` in the project. Use persistent storage in production. |

The AI chat endpoint calls Groq from the server. The current server does not read a documented Groq credential environment variable, so verify and update that integration to use the deployment's secret-management mechanism before production. Do not expose API credentials in browser code or committed files.

## Pages and routes

The Express server maps short routes to the corresponding HTML pages. Static files (scripts, styles, translations, and images) are served from the project root.

| URL | Page |
| --- | --- |
| `/` | Home |
| `/aboutus` | About Us |
| `/services` | Services |
| `/admission` | Admissions information |
| `/apply` | Submit admission documents |
| `/visit` | Request a campus visit |
| `/news` | News |
| `/admin` | News administration |
| `/gallery` | Gallery |
| `/chatbot` | AI assistant |
| `/student-login` | Student sign-in and registration |
| `/courses` | Student course portal |
| `/courses-admin` | Instructor course and student management |

The source HTML files can also be reached as static paths through the Express static-file middleware.

## API overview

All endpoints are served by the same Express app.

### News

| Method and path | Purpose | Access |
| --- | --- | --- |
| `POST /api/auth/login` | Exchange the news administrator password for a bearer token. | Public login |
| `GET /api/posts` | List posts, newest first; accepts an optional `category` query parameter. | Public |
| `GET /api/posts/:id` | Read one post. | Public |
| `GET /api/categories` | List distinct post categories. | Public |
| `POST /api/posts` | Create a post with English/optional Arabic text and an optional image upload. | News administrator |
| `DELETE /api/posts/:id` | Delete a post and its image when present. | News administrator |

### Courses and students

| Method and path | Purpose | Access |
| --- | --- | --- |
| `POST /api/courses/instructor-login` | Instructor login. | Public login |
| `POST /api/courses/student-register` | Create a student account. | Public registration |
| `POST /api/courses/student-login` | Student login. | Public login |
| `GET /api/courses/grades` | List course grades. | Student or instructor token |
| `GET /api/courses` | List courses; accepts an optional `grade` query parameter. | Student or instructor token |
| `POST /api/courses` | Create a course. | Instructor |
| `PUT /api/courses/:id` | Update a course. | Instructor |
| `DELETE /api/courses/:id` | Delete a course. | Instructor |
| `GET /api/students` | List students without password hashes. | Instructor |
| `PUT /api/students/:id` | Update a student's name, username, grade, and optionally password. | Instructor |
| `DELETE /api/students/:id` | Delete a student account. | Instructor |

Protected API requests use an `Authorization: Bearer <token>` header. Student passwords are hashed before being stored. Course videos and PDF materials are links entered by instructors; course media is not uploaded by this app.

### Forms and AI

| Method and path | Purpose |
| --- | --- |
| `POST /send-admission` | Accepts multipart form data (`name`, `email`, `phone`, and `file`) and emails the document. Files must be PDF, ZIP, or RAR and are limited to 10 MB. |
| `POST /send-visit` | Accepts JSON (`name`, `phone`, `date`, and optional `notes`) and emails a visit request. |
| `POST /chat` | Accepts a message and optional conversation history, then returns an AI-generated reply. |

## Data and uploads

MongoDB stores news posts, courses, and student accounts. The server connects to the database asynchronously when it starts; database-backed features require a reachable MongoDB service.

News images are stored on disk in `News/uploads/` by default and are served at `/uploads/...`. Set `UPLOADS_DIR` to a persistent volume path in hosted deployments so uploaded images survive redeploys. The admission form's file is held in memory for sending as an email attachment; it is not saved to the upload directory.

The `News/uploads/` and `Server/uploads/` directories are ignored by Git except for their `.gitkeep` placeholders. Back up or persist uploaded content separately if it must be retained.

## Deployment

`railway.json` configures the Nixpacks builder and starts the app with `cd Server && node server.js`. It also enables restart-on-failure with up to 10 retries.

For a hosted deployment:

1. Configure the MongoDB connection, Resend credential, admin/instructor passwords, and CORS origin in the hosting platform's environment settings. Store the Groq credential as a deployment secret and update the current chat integration to read it securely.
2. Configure `UPLOADS_DIR` to point to persistent storage and mount that storage at the chosen path.
3. Deploy from the repository root; Railway uses the configured start command.
4. Check server logs and verify database, email, and AI integrations using their corresponding pages.

## Development notes

- The root `package.json` provides `npm start` and installs the server package through `postinstall`; the Express-specific development command is `npm run dev` from `Server/`.
- No automated test, lint, front-end build, or SCSS compile scripts are currently declared in the package manifests.
- Most pages share the language toggle and navigation pattern, while the news, courses, admin, and chatbot pages have page-specific scripts.
- Arabic page direction and translations are driven by `i18n.js`; news and course records also support optional Arabic fields.
