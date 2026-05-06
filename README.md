#!/usr/bin/env python3
"""
Lightweight Linux MP3/WAV Player Backend

This script scans the music folder, plays MP3 files using mpg123,
plays WAV files using aplay, and connects button input to playback functions.

For demo mode, use_mock_buttons is enabled in config.json.
That lets the script run without real GPIO buttons connected.
"""

import json
import subprocess
from pathlib import Path

CONFIG_FILE = Path(__file__).parent / "config.json"


class MP3Player:
    def __init__(self):
        self.config = self.load_config()
        self.music_library = Path(self.config["music_library"])
        self.supported_formats = tuple(self.config["supported_formats"])
        self.songs = self.scan_music()
        self.current_index = 0
        self.process = None

    def load_config(self):
        with open(CONFIG_FILE, "r") as file:
            return json.load(file)

    def scan_music(self):
        songs = []
        if not self.music_library.exists():
            print(f"Music library not found: {self.music_library}")
            return songs

        for path in self.music_library.rglob("*"):
            if path.suffix.lower() in self.supported_formats:
                songs.append(path)

        songs.sort()
        print(f"Found {len(songs)} song(s).")
        return songs

    def current_song(self):
        if not self.songs:
            return None
        return self.songs[self.current_index]

    def play(self):
        song = self.current_song()
        if song is None:
            print("No songs found.")
            return

        self.stop()

        if song.suffix.lower() == ".mp3":
            command = ["mpg123", str(song)]
        elif song.suffix.lower() == ".wav":
            command = ["aplay", str(song)]
        else:
            print(f"Unsupported file type: {song}")
            return

        print(f"Now playing: {song.name}")
        self.process = subprocess.Popen(command)
        self.update_display(f"Playing: {song.name}")

    def pause(self):
        self.stop()
        self.update_display("Paused")

    def stop(self):
        if self.process and self.process.poll() is None:
            self.process.terminate()
            try:
                self.process.wait(timeout=2)
            except subprocess.TimeoutExpired:
                self.process.kill()
        self.process = None

    def next_song(self):
        if not self.songs:
            print("No songs available.")
            return
        self.current_index = (self.current_index + 1) % len(self.songs)
        self.play()

    def previous_song(self):
        if not self.songs:
            print("No songs available.")
            return
        self.current_index = (self.current_index - 1) % len(self.songs)
        self.play()

    def volume_up(self):
        print("Volume up")
        subprocess.run(["amixer", "set", "Master", "5%+"], check=False)

    def volume_down(self):
        print("Volume down")
        subprocess.run(["amixer", "set", "Master", "5%-"], check=False)

    def update_display(self, message):
        # Placeholder for OLED/LCD display code.
        print(f"[DISPLAY] {message}")

    def mock_button_loop(self):
        print("\nDemo keyboard controls:")
        print("p = play")
        print("s = stop/pause")
        print("n = next")
        print("b = previous")
        print("+ = volume up")
        print("- = volume down")
        print("q = quit")

        while True:
            choice = input("\nEnter command: ").strip().lower()

            if choice == "p":
                self.play()
            elif choice == "s":
                self.pause()
            elif choice == "n":
                self.next_song()
            elif choice == "b":
                self.previous_song()
            elif choice == "+":
                self.volume_up()
            elif choice == "-":
                self.volume_down()
            elif choice == "q":
                self.stop()
                print("Exiting player.")
                break
            else:
                print("Invalid command.")

    def gpio_button_loop(self):
        try:
            from gpiozero import Button
            from signal import pause
        except ImportError:
            print("gpiozero is not installed. Run: pip3 install gpiozero")
            return

        pins = self.config["gpio_pins"]

        play_pause_button = Button(pins["play_pause"])
        next_button = Button(pins["next"])
        previous_button = Button(pins["previous"])
        volume_up_button = Button(pins["volume_up"])
        volume_down_button = Button(pins["volume_down"])

        play_pause_button.when_pressed = self.play
        next_button.when_pressed = self.next_song
        previous_button.when_pressed = self.previous_song
        volume_up_button.when_pressed = self.volume_up
        volume_down_button.when_pressed = self.volume_down

        print("GPIO button mode running.")
        pause()

    def run(self):
        if self.config.get("use_mock_buttons", True):
            self.mock_button_loop()
        else:
            self.gpio_button_loop()


if __name__ == "__main__":
    MP3Player().run()

