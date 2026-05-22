# Build Instructions
[Back to Home](../00_Home.md)

## Web Application (Local)

Project root: `Web (flask)/`

1. Create and activate a Python virtual environment.
2. Install dependencies from `requirements.txt`.
3. Configure database URI and mail credentials as environment variables.
4. Run Flask app (`app.py`) and initialize DB tables.

## Suggested Environment Variables

| Variable | Purpose |
|---|---|
| `SECRET_KEY` | Flask session signing key |
| `DATABASE_URL` | SQLAlchemy connection string |
| `MAIL_USERNAME` | SMTP sender identity |
| `MAIL_PASSWORD` | SMTP app password |
| `TEXTBEE_API_KEY` | SMS API authentication |
| `TEXTBEE_DEVICE_ID` | SMS gateway device selector |

## Docker Build

The repository includes `Web (flask)/Dockerfile`.

```bash
docker build -t Lumyn-web "Web (flask)"
docker run -p 5000:5000 --env-file .env Lumyn-web
```

## Deployment Notes

- `vercel.json` is configured to build from Docker.
- Ensure secrets are injected by platform environment configuration.

## Dependency List

| Package | Version |
|---|---|
| Flask | 3.0.3 |
| Flask-SQLAlchemy | 3.0.3 |
| Flask-Login | 0.6.3 |
| Flask-Mail | 0.9.3 |
| PyMySQL | 1.1.0 |
| gunicorn | unpinned |

See [Software-Architecture]().
