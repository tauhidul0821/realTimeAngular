# Real-Time Company Logo Update in Angular with Node.js and MongoDB

## Overview
This guide explains how to implement real-time company logo updates in an Angular project using a Node.js backend with MongoDB. The solution leverages **Socket.IO** for real-time updates so that all connected users see the new logo immediately without refreshing the page.

---

## 1. Backend Setup (Node.js + MongoDB)
### Install Dependencies
```sh
npm install express mongoose cors socket.io
```

### Server Code (`server.js`)
```javascript
const express = require("express");
const http = require("http");
const mongoose = require("mongoose");
const cors = require("cors");
const socketIo = require("socket.io");

const app = express();
const server = http.createServer(app);
const io = socketIo(server, { cors: { origin: "*" } });

app.use(cors());
app.use(express.json());

// Connect to MongoDB
mongoose.connect("mongodb://localhost:27017/mydb", {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

// Define a schema and model
const CompanySchema = new mongoose.Schema({ logoUrl: String });
const Company = mongoose.model("Company", CompanySchema);

// Socket.IO connection
io.on("connection", (socket) => {
  console.log("Client connected:", socket.id);
});

// API to update the logo
app.put("/api/company/logo", async (req, res) => {
  const { logoUrl } = req.body;
  await Company.findOneAndUpdate({}, { logoUrl }, { upsert: true });
  io.emit("logo-updated", logoUrl);
  res.json({ message: "Logo updated successfully", logoUrl });
});

// API to get the logo (for initial load)
app.get("/api/company/logo", async (req, res) => {
  const company = await Company.findOne();
  res.json({ logoUrl: company?.logoUrl || "" });
});

server.listen(3000, () => console.log("Server running on port 3000"));
```

---

## 2. Angular Frontend Setup
### Install Dependencies
```sh
npm install ngx-socket-io
```

### Create Socket Service (`logo.service.ts`)
```typescript
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Socket } from "ngx-socket-io";
import { BehaviorSubject } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class LogoService {
  private logoSubject = new BehaviorSubject<string>("");
  logo$ = this.logoSubject.asObservable();

  constructor(private http: HttpClient, private socket: Socket) {
    // Fetch the initial logo
    this.http.get<{ logoUrl: string }>("/api/company/logo").subscribe((data) => {
      this.logoSubject.next(data.logoUrl);
    });

    // Listen for real-time updates
    this.socket.fromEvent<string>("logo-updated").subscribe((newLogo) => {
      this.logoSubject.next(newLogo);
    });
  }

  updateLogo(newLogoUrl: string) {
    this.http.put<{ message: string; logoUrl: string }>("/api/company/logo", { logoUrl: newLogoUrl })
      .subscribe((response) => {
        console.log(response.message);
      });
  }
}
```

### Display Logo in Header Component (`header.component.ts`)
```typescript
import { Component } from "@angular/core";
import { LogoService } from "../services/logo.service";

@Component({
  selector: "app-header",
  template: `<img [src]="logo$ | async" alt="Company Logo" width="100">`,
})
export class HeaderComponent {
  logo$ = this.logoService.logo$;

  constructor(private logoService: LogoService) {}
}
```

### Update Logo from Profile Component (`profile.component.ts`)
```typescript
import { Component } from "@angular/core";
import { LogoService } from "../services/logo.service";

@Component({
  selector: "app-profile",
  template: `
    <input type="text" [(ngModel)]="newLogoUrl" placeholder="Enter logo URL">
    <button (click)="changeLogo()">Update Logo</button>
  `,
})
export class ProfileComponent {
  newLogoUrl: string = "";

  constructor(private logoService: LogoService) {}

  changeLogo() {
    this.logoService.updateLogo(this.newLogoUrl);
  }
}
```

---

## 3. How It Works
1. **User updates the logo** in `ProfileComponent`.
2. The **logo is saved in MongoDB** and **emitted via Socket.IO**.
3. **All connected browsers (Chrome, Firefox, etc.) receive the update instantly** and update the displayed logo.
4. **When a new user opens the app**, they get the latest logo from MongoDB.

---

## 4. Expected Behavior
✅ **Mozilla:** Updates the logo → Chrome sees the change immediately.  
✅ **Chrome:** Updates the logo → Mozilla sees the change immediately.  
✅ **Other Browsers:** Any browser connected to the server will receive the update in real time.  
✅ **No Page Refresh Needed!** 🚀

---

## 5. Alternative: Firebase Firestore for Real-Time Updates
If you don't want to manage WebSockets, use Firebase Firestore:
```typescript
this.firestore
  .collection("company")
  .doc("logo")
  .valueChanges()
  .subscribe((data) => {
    this.store.dispatch(updateLogo({ logoUrl: data.logoUrl }));
  });
```

---

## 6. Conclusion
Using **Socket.IO** with **Node.js and MongoDB**, we ensure that **any change to the company logo is instantly reflected across all users in real time**, without requiring a page refresh.

🚀 **Now, users in different browsers and devices see changes instantly!**

---
Let me know if you have any questions! 😊



# 📍 Real-Time Location Tracking in Angular Admin Panel

## 🚀 Overview
This guide explains how to implement **real-time delivery tracking** in an **Angular admin panel**, showing both:
- **Customer's house location** (fixed)
- **Delivery person's live location** (updates in real-time)

## 🛠️ Technologies Used
- **Angular** (Frontend Framework)
- **Google Maps API** (Display locations)
- **WebSockets (Socket.io)** (Real-time updates)
- **Geolocation API** (Track driver location)
- **Node.js + Express** (Backend for WebSockets)

---

## 📌 1. Install Dependencies

### 🔹 Install Google Maps for Angular
```sh
npm install @angular/google-maps
```

Enable **Maps JavaScript API** in [Google Cloud Console](https://console.cloud.google.com/).

### 🔹 Install Socket.io Client
```sh
npm install socket.io-client
```

---

## 📌 2. Create Angular Service for Location Updates

**`location.service.ts`**
```typescript
import { Injectable } from '@angular/core';
import { io } from 'socket.io-client';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class LocationService {
  private socket = io('https://yourserver.com'); // Change to your backend URL

  sendDriverLocation(driverId: string, lat: number, lng: number) {
    this.socket.emit('driverLocation', { driverId, lat, lng });
  }

  getDriverLocation(): Observable<any> {
    return new Observable((observer) => {
      this.socket.on('updateLocation', (data) => {
        observer.next(data);
      });
    });
  }
}
```

---

## 📌 3. Display Google Map in Angular Admin Panel

**`admin-map.component.ts`**
```typescript
import { Component, OnInit } from '@angular/core';
import { LocationService } from '../services/location.service';

@Component({
  selector: 'app-admin-map',
  templateUrl: './admin-map.component.html',
  styleUrls: ['./admin-map.component.css']
})
export class AdminMapComponent implements OnInit {
  customerLocation = { lat: 40.7128, lng: -74.0060 }; // Example: New York
  driverLocation = { lat: 0, lng: 0 };
  
  constructor(private locationService: LocationService) {}

  ngOnInit(): void {
    this.locationService.getDriverLocation().subscribe((data) => {
      this.driverLocation = { lat: data.lat, lng: data.lng };
    });
  }
}
```

---

## 📌 4. Show Google Maps in Admin Panel UI

**`admin-map.component.html`**
```html
<google-map [center]="customerLocation" [zoom]="14" height="400px" width="100%">
  <map-marker [position]="customerLocation" title="Customer House"></map-marker>
  <map-marker [position]="driverLocation" title="Delivery Man" icon="https://maps.google.com/mapfiles/ms/icons/blue-dot.png"></map-marker>
</google-map>
```

---

## 📌 5. Backend (Node.js + Socket.io) for Real-Time Updates

**`server.js`**
```javascript
const io = require("socket.io")(3000, {
  cors: {
    origin: "*",
  },
});

io.on("connection", (socket) => {
  console.log("New Admin Connected");

  socket.on("driverLocation", (data) => {
    console.log("Driver Location:", data);
    io.emit("updateLocation", data); // Broadcast to all admins
  });
});
```

---

## 🎯 Features You Get
✅ **Real-time delivery tracking** on the **admin panel**  
✅ **Customer house location** is fixed  
✅ **Driver’s live location** updates automatically  
✅ **WebSockets (fast, no delays)**  
✅ **Google Maps for accurate visualization**  

---

## 📌 Future Enhancements
- **🚗 Route Display**: Show the path from the driver to the customer.
- **⏳ ETA Calculation**: Predict delivery time.
- **📍 Multiple Deliveries**: Track multiple drivers at once.

---

### 🎉 Done! Your Angular admin panel now has real-time delivery tracking! 🚀

