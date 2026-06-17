SHEHaven Bedding Unified Portal - User Type Tabs Build

This build keeps the working SHEHaven stock/admin system and changes the first screen into clear tabs:

- Client
- Our Employee

Client goes to the client order page.
Our Employee goes to the employee login page.

Client orders still save into the same Supabase database and appear in the employee/user platform under Client Orders as Pending orders.

Email and WhatsApp notifications are not included in this build; they can be added later.

Upload these files only to GitHub root:
- app.py
- requirements.txt
- runtime.txt
- README.txt

Streamlit Secret required:
SUPABASE_DB_URL = "your Supabase database URL"

Default employee login:
Username: admin
Password: admin123
