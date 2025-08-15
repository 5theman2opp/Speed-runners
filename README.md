# Speed-runners
This is for the app I am try to make that will inhanse a persons speed by giving food back about their form, speed, and what they need to do to improve on it.

import React, { useState, useEffect, useRef } from 'react';
import { StyleSheet, Text, View, Button, TextInput, Platform } from 'react-native';
import * as Location from 'expo-location';
import * as Speech from 'expo-speech';
import { activateKeepAwake, deactivateKeepAwake } from 'expo-keep-awake';

export default function App() {
  const [running, setRunning] = useState(false);
  const [distance, setDistance] = useState(0);
  const [speed, setSpeed] = useState(0);
  const [acceleration, setAcceleration] = useState(0);
  const [prevSpeed, setPrevSpeed] = useState(0);
  const [targetPace, setTargetPace] = useState('5:30');
  const [lastCoords, setLastCoords] = useState(null);
  const watchId = useRef(null);
  const timerRef = useRef(null);

  const paceToSecPerKm = (pace) => {
    const [min, sec] = pace.split(':').map(Number);
    return min * 60 + sec;
  };

  const getPaceFromSpeed = (spd) => {
    if (spd <= 0) return 0;
    return 1000 / spd; // seconds per km
  };

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
    return R * c; // meters
  };

  const startRun = async () => {
    let { status } = await Location.requestForegroundPermissionsAsync();
    if (status !== 'granted') {
      alert('Location permission is required.');
      return;
    }
    setDistance(0);
    setSpeed(0);
    setAcceleration(0);
    setPrevSpeed(0);
    setLastCoords(null);
    setRunning(true);
    activateKeepAwake();

    watchId.current = await Location.watchPositionAsync(
      { accuracy: Location.Accuracy.High, timeInterval: 1000, distanceInterval: 1 },
      (loc) => {
        const coords = loc.coords;
        if (lastCoords) {
          const dist = haversineDistance(lastCoords, coords);
          setDistance((prev) => prev + dist);
        }
        setLastCoords(coords);

        const spd = coords.speed || 0; // m/s
        setSpeed(spd);
        const acc = (spd - prevSpeed) / 1; // per second
        setAcceleration(acc);
        setPrevSpeed(spd);
      }
    );

    timerRef.current = setInterval(() => {
      const currentPaceSec = getPaceFromSpeed(speed);
      const targetSec = paceToSecPerKm(targetPace);
      if (currentPaceSec === 0) return;
      let feedback = '';
      if (currentPaceSec < targetSec - 10) feedback = 'Ease up, you are ahead of pace';
      else if (currentPaceSec > targetSec + 10) feedback = 'Speed up, you are behind pace';
      else feedback = 'Good job, on pace';
      Speech.speak(`${feedback}. Distance ${(distance / 1000).toFixed(2)} km`);
    }, 60000);
  };

  const stopRun = () => {
    if (watchId.current) {
      watchId.current.remove();
      watchId.current = null;
    }
    clearInterval(timerRef.current);
    timerRef.current = null;
    setRunning(false);
    deactivateKeepAwake();
  };

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

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 20 },
  header: { fontSize: 28, fontWeight: 'bold', marginBottom: 20 },
  input: {
    borderWidth: 1,
    borderColor: '#ccc',
    padding: 10,
    width: 150,
    marginBottom: 20,
    textAlign: 'center',
  },
});
