# Speed-runners App

## 🏃 About The App

Speed-runners is a mobile application designed to help runners improve their performance. By providing real-time feedback on **speed, pace, and acceleration**, the app acts as a personal running coach. It uses your device's GPS to track your run and a text-to-speech feature to give you audio cues and encouragement to help you stay on track with your target pace.

## ✨ Features

* **Real-time Tracking**: Monitors your distance, speed, and acceleration as you run.

* **Target Pace Setting**: Allows you to set a desired pace (e.g., 5:30 per km) to guide your workout.

* **Audio Feedback**: Gives you verbal updates every minute, telling you if you are ahead of, behind, or on pace.

* **Screen Awake**: Keeps your screen on during the run so you can easily check your stats.

## 🛠️ How It Works

The app uses the Haversine formula to accurately calculate the distance between GPS coordinates. It then uses this data, along with timestamps, to determine your current speed and acceleration. All of this information is used to provide timely and relevant feedback.

## ⚙️ Getting Started

### Prerequisites

* Expo Go app installed on your device

* Node.js and npm/yarn installed on your machine

### Installation

1. Clone this repository.

2. Navigate to the project directory.

3. Install the required dependencies:# Speed-runners
This is for the app I am try to make that will inhanse a persons speed by giving food back about their form, speed, and what they need to do to improve on it.



























import React, { useState, useEffect, useRef } from "react";
import { StyleSheet, Text, View, Button, TextInput, Alert } from "react-native";
import * as Location from "expo-location";
import * as Speech from "expo-speech";
import { activateKeepAwake, deactivateKeepAwake } from "expo-keep-awake";

// =====================
// Utility Functions
// =====================

// Convert mm:ss pace to total seconds
const paceToSecondsPerKm = (pace) => {
  const [min, sec] = pace.split(":").map(Number);
  return min * 60 + sec;
};

// Convert speed (m/s) to pace (s/km)
const getPaceFromSpeed = (speed) => (speed > 0 ? 1000 / speed : 0);

// Calculate distance in meters between two GPS coords
const haversineDistance = (coords1, coords2) => {
  const toRad = (x) => (x * Math.PI) / 180;
  const R = 6371e3; // meters
  const lat1 = toRad(coords1.latitude);
  const lat2 = toRad(coords2.latitude);
  const dLat = toRad(coords2.latitude - coords1.latitude);
  const dLon = toRad(coords2.longitude - coords1.longitude);
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) ** 2;
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;
};

export default function App() {
  const [running, setRunning] = useState(false);
  const [distance, setDistance] = useState(0);
  const [speed, setSpeed] = useState(0);
  const [acceleration, setAcceleration] = useState(0);
  const [targetPace, setTargetPace] = useState("5:30");
  const [lastCoords, setLastCoords] = useState(null);
  const [prevSpeed, setPrevSpeed] = useState(0);
  const [prevTimestamp, setPrevTimestamp] = useState(null);

  const watchId = useRef(null);
  const timerRef = useRef(null);

  // =====================
  // Start Run
  // =====================
  const startRun = async () => {
    const { status } = await Location.requestForegroundPermissionsAsync();
    if (status !== "granted") {
      Alert.alert("Permission Required", "Location access is required to track runs.");
      return;
    }

    // Reset session
    setDistance(0);
    setSpeed(0);
    setAcceleration(0);
    setPrevSpeed(0);
    setPrevTimestamp(null);
    setLastCoords(null);
    setRunning(true);
    activateKeepAwake();

    // Start GPS tracking
    watchId.current = await Location.watchPositionAsync(
      { accuracy: Location.Accuracy.High, timeInterval: 1000, distanceInterval: 1 },
      (loc) => {
        const coords = loc.coords;

        // Distance tracking
        if (lastCoords) {
          const dist = haversineDistance(lastCoords, coords);
          setDistance((prev) => prev + dist);
        }
        setLastCoords(coords);

        // Speed (m/s)
        const currentSpeed = coords.speed || 0;
        setSpeed(currentSpeed);

        // Acceleration (m/s²) with time difference
        if (prevTimestamp) {
          const timeDiff = (loc.timestamp - prevTimestamp) / 1000;
          if (timeDiff > 0) {
            setAcceleration((currentSpeed - prevSpeed) / timeDiff);
          }
        }

        setPrevSpeed(currentSpeed);
        setPrevTimestamp(loc.timestamp);
      }
    );

    // Speech feedback every minute
    timerRef.current = setInterval(() => {
      givePaceFeedback();
    }, 60000);
  };

  // =====================
  // Stop Run
  // =====================
  const stopRun = () => {
    if (watchId.current) {
      watchId.current.remove();
      watchId.current = null;
    }
    if (timerRef.current) {
      clearInterval(timerRef.current);
      timerRef.current = null;
    }
    deactivateKeepAwake();
    setRunning(false);
  };

  // =====================
  // Pace Feedback
  // =====================
  const givePaceFeedback = () => {
    const currentPaceSec = getPaceFromSpeed(speed);
    if (currentPaceSec === 0) return;

    const targetSec = paceToSecondsPerKm(targetPace);
    let feedback;

    if (currentPaceSec < targetSec - 10) {
      feedback = "Ease up, you are ahead of pace.";
    } else if (currentPaceSec > targetSec + 10) {
      feedback = "Speed up, you are behind pace.";
    } else {
      feedback = "Good job, you are on pace.";
    }

    Speech.speak(`${feedback} Distance ${(distance / 1000).toFixed(2)} kilometers.`);
  };

  // =====================
  // UI
  // =====================
  return (
    <View style={styles.container}>
      <Text style={styles.header}>Runner Tracker</Text>

      <TextInput
        style={styles.input}
        value={targetPace}
        onChangeText={setTargetPace}
        placeholder="Target pace (mm:ss)"
      />

      <Text>Distance: {(distance / 1000).toFixed(2)} km</Text>
      <Text>Speed: {(speed * 3.6).toFixed(2)} km/h</Text>
      <Text>Acceleration: {acceleration.toFixed(2)} m/s²</Text>

      {!running ? (
        <Button title="Start Run" onPress={startRun} />
      ) : (
        <Button title="Stop Run" onPress={stopRun} color="red" />
      )}
    </View>
  );
}

// =====================
// Styles
// =====================
const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: "center",
    alignItems: "center",
    padding: 20,
    backgroundColor: "#f5f5f5",
  },
  header: {
    fontSize: 28,
    fontWeight: "bold",
    marginBottom: 20,
  },
  input: {
    borderWidth: 1,
    borderColor: "#ccc",
    padding: 10,
    width: 150,
    marginBottom: 20,
    textAlign: "center",
    borderRadius: 5,
    backgroundColor: "#fff",
  },
});

  },
});
