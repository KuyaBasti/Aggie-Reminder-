# Aggie Reminder

A comprehensive volunteer management system designed to help administrators efficiently manage volunteer schedules, track work hours, and send automated reminders. Built for HackDavis 2024.

## 🌟 Features

### Web Application
- **Volunteer Hour Management**: Track and manage volunteer work hours through an intuitive web interface
- **Email Communication**: Send reminders and notifications to volunteers using SendGrid API
- **Admin Dashboard**: Centralized interface for managing volunteers and their schedules
- **Database Integration**: PostgreSQL backend for data persistence
- **Excel Integration**: Import and work with Excel files for volunteer data management

### Automated Reminder System
- **Google Apps Script Integration**: Automated reminder system using Google Sheets
- **Smart Scheduling**: Automatically sends reminders 2 days before events
- **Email Automation**: Time-driven triggers for seamless volunteer communication
- **Easy Configuration**: Simple setup process with Google Sheets interface

## 🛠️ Tech Stack

**Backend:**
- Node.js
- Express.js
- PostgreSQL
- Knex.js (Query Builder)

**Frontend:**
- HTML5
- CSS3
- JavaScript
- Responsive Design

**APIs & Services:**
- SendGrid Email API
- Google Apps Script
- Google Sheets API

**Additional Tools:**
- XLSX (Excel file processing)
- Nodemon (Development)
- CORS (Cross-Origin Resource Sharing)

## 📁 Project Structure

```
Aggie-Reminder/
├── hackdavis2024/              # Main web application
│   ├── public/                 # Static web assets
│   │   ├── css/               # Stylesheets
│   │   ├── js/                # Client-side JavaScript
│   │   ├── img/               # Images
│   │   ├── index.html         # Main dashboard
│   │   ├── login.html         # Login page
│   │   └── register.html      # Registration page
│   ├── server.js              # Express server
│   ├── package.json           # Dependencies and scripts
│   └── Test Calendar 2024.xlsx # Sample data file
├── automated_reminder/         # Google Apps Script automation
│   ├── Formatted Sheet.xlsx   # Template spreadsheet
│   ├── automated_reminder.txt # Apps Script code
│   └── README!.txt           # Setup instructions
├── Procedures.md              # Detailed setup guide
└── README.md                 # This file
```

## 🚀 Installation

### Prerequisites

- **Node.js** (v14 or higher) - [Download here](https://nodejs.org/en)
- **PostgreSQL** - [Installation guide in Procedures.md](./Procedures.md)
- **SendGrid Account** - For email functionality

### Quick Start

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-username/Aggie-Reminder.git
   cd Aggie-Reminder
   ```

2. **Navigate to the main application**
   ```bash
   cd hackdavis2024
   ```

3. **Install dependencies**
   ```bash
   npm install
   ```

4. **Configure the application**
   - Edit `server.js` line 9 and add your Excel file path:
     ```javascript
     const filePath = '/path/to/your/excel/file.xlsx'; // Updated file path
     ```
   - Add your SendGrid API key (see Configuration section)

5. **Start the server**
   ```bash
   npm start
   # or
   node server.js
   ```

6. **Access the application**
   Open your browser and navigate to `http://localhost:3000`

## ⚙️ Configuration

### SendGrid Email Setup

1. Create a [SendGrid account](https://sendgrid.com)
2. Generate an API key from your SendGrid dashboard
3. Add the API key to your server configuration
4. Configure sender email verification

### Database Setup

Follow the detailed PostgreSQL installation guide in [Procedures.md](./Procedures.md) for complete database setup instructions.

### Google Apps Script Automation

1. **Upload the spreadsheet**
   - Download `Formatted Sheet.xlsx` from the `automated_reminder/` folder
   - Upload to Google Sheets

2. **Set up Apps Script**
   - Go to Extensions → Apps Script
   - Copy code from `automated_reminder.txt`
   - Paste into Code.gs

3. **Configure triggers**
   - Go to Triggers → Add Trigger
   - Set event source to "Time-driven"
   - Set trigger type to "Day timer"
   - Choose your preferred reminder time
   - Save

## 📖 Usage

### For Administrators

1. **Login/Register**: Access the admin dashboard through the login page
2. **Manage Volunteers**: View and edit volunteer information and schedules
3. **Track Hours**: Monitor volunteer work hours and attendance
4. **Send Reminders**: Use the interface to send manual reminders via email
5. **Excel Integration**: Import volunteer data from Excel files

### For Automated Reminders

1. **Update Google Sheet**: Add volunteer emails and event details
2. **Set Triggers**: Configure when reminders should be sent
3. **Monitor**: The system automatically sends reminders 2 days before events

## 🔧 Development

### Running in Development Mode

```bash
npm start  # Uses nodemon for auto-restart
```

### Dependencies

Key dependencies include:
- `express`: Web framework
- `@sendgrid/mail`: Email service
- `pg`: PostgreSQL client
- `xlsx`: Excel file processing
- `knex`: SQL query builder
- `cors`: Cross-origin resource sharing

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the ISC License - see the [package.json](hackdavis2024/package.json) file for details.

## 🏆 Acknowledgments

- Built for HackDavis 2024
- Created to enhance volunteer management for Aggie House
- Special thanks to the development team for their dedication and collaboration

## 🐛 Troubleshooting

### Common Issues

- **Port 3000 already in use**: Kill existing processes or change the port in server.js
- **Database connection issues**: Ensure PostgreSQL is running and configured correctly
- **Email not sending**: Verify SendGrid API key and sender verification
- **Excel file not loading**: Check file path in server.js line 9

For detailed setup instructions, see [Procedures.md](./Procedures.md).

## 📞 Support

If you encounter any issues or have questions, please:
1. Check the [Procedures.md](./Procedures.md) file for detailed setup instructions
2. Review the troubleshooting section above
3. Open an issue on GitHub

---

**Note**: This project requires proper API key configuration for full functionality. Ensure all prerequisites are met before deployment.
