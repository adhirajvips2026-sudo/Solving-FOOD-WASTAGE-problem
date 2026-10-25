# Solving-FOOD-WASTAGE-problem
Smart Silo helps protect stored grain from weather damage. Upload a weather screenshot and the program reads the temperature, humidity and rainfall using OCR. It then gives a risk level (Low, Moderate or High) with simple actions to take. Made with Python and Google Colab. A first-check tool, not a final test.
# SMART SILO WEATHER IMAGE ANALYZER
# Run in Google Colab

import subprocess
import sys
import re
import os

# 1. Install OCR tools
print("Setting up image-reading software...")
subprocess.run(
    ["apt-get", "update", "-qq"],
    check=True
)
subprocess.run(
    ["apt-get", "install", "-y", "-qq", "tesseract-ocr"],
    check=True
)
subprocess.run(
    [sys.executable, "-m", "pip", "install", "-q",
     "pytesseract", "pillow"],
    check=True
)

import pytesseract
from PIL import Image, ImageEnhance, ImageFilter
from google.colab import files

# 2. Upload a weather screenshot or image
print("\nUpload your weather screenshot (JPG, PNG, etc.)")
uploaded = files.upload()

if not uploaded:
    raise ValueError("No image uploaded.")

filename = next(iter(uploaded))

# 3. Read the image using OCR
img = Image.open(filename).convert("RGB")
img = img.resize((img.width * 2, img.height * 2))
img = ImageEnhance.Contrast(img).enhance(1.5)
img = img.filter(ImageFilter.SHARPEN)

ocr_text = pytesseract.image_to_string(img)
print("\nTEXT READ FROM YOUR IMAGE:")
print("-" * 40)
print(ocr_text if ocr_text.strip() else
      "No readable text found. Try a clearer screenshot.")

# 4. Extract values near recognizable labels
def extract_value(patterns, text):
    for pattern in patterns:
        match = re.search(
            pattern, text, re.IGNORECASE | re.MULTILINE
        )
        if match:
            try:
                return float(match.group(1).replace(",", "."))
            except ValueError:
                pass
    return None

temperature = extract_value([
    r"(?:temperature|temp|feels like)\s*[:=]?\s*"
    r"(-?\d{1,2}(?:\.\d+)?)\s*°?\s*c\b",
    r"(-?\d{1,2}(?:\.\d+)?)\s*°\s*c\b"
], ocr_text)

humidity = extract_value([
    r"(?:relative humidity|humidity|humid(?:ity)?)"
    r"\s*[:=]?\s*(\d{1,3})\s*%"
], ocr_text)

rainfall = extract_value([
    r"(?:rainfall|precipitation|rain)\s*[:=]?\s*"
    r"(\d+(?:\.\d+)?)\s*mm\b"
], ocr_text)

print("\nEXTRACTED WEATHER VALUES")
print("Temperature:", temperature)
print("Humidity:", humidity)
print("Rainfall:", rainfall)

# 5. Confirm values; OCR may misread screenshots
def get_value(label, extracted, unit, minimum, maximum):
    if extracted is not None and minimum <= extracted <= maximum:
        answer = input(
            f"{label} detected as {extracted}{unit}. "
            "Press Enter to accept, or type a correction: "
        ).strip()
        if answer == "":
            return extracted
    else:
        print(f"Could not confidently read {label} from the image.")

    while True:
        answer = input(
            f"Enter {label} in {unit} (or type 'unknown'): "
        ).strip()

        if answer.lower() == "unknown":
            return None

        try:
            value = float(answer)
            if minimum <= value <= maximum:
                return value
            print(f"Enter a value between {minimum} and {maximum}.")
        except ValueError:
            print("Please enter a number or 'unknown'.")

temperature = get_value(
    "Outside temperature", temperature, "°C", -50, 60
)
humidity = get_value(
    "Relative humidity", humidity, "%", 0, 100
)
rainfall = get_value(
    "Rainfall", rainfall, "mm", 0, 2000
)

location = input(
    "\nEnter the location shown in the weather report: "
).strip() or "Location not specified"

grain = input(
    "Enter stored grain type (rice/wheat/maize/etc.): "
).strip() or "Grain type not specified"

# Optional measurements from the silo itself
moisture_input = input(
    "Grain moisture percentage, if measured (Enter to skip): "
).strip()

silo_temp_input = input(
    "Internal silo temperature in °C, if measured (Enter to skip): "
).strip()

moisture = float(moisture_input) if moisture_input else None
silo_temp = float(silo_temp_input) if silo_temp_input else None

if moisture is not None and not 0 <= moisture <= 100:
    raise ValueError("Grain moisture must be between 0 and 100.")

if silo_temp is not None and not -50 <= silo_temp <= 70:
    raise ValueError("Check the internal silo temperature.")

# 6. Analyze the conditions
risks = []
actions = []
score = 0

if humidity is not None and humidity >= 80:
    risks.append("High outside humidity")
    actions.append(
        "Inspect seals and check for condensation and grain moisture."
    )
    score += 1

if temperature is not None and temperature >= 35:
    risks.append("High outside temperature")
    actions.append(
        "Monitor grain temperature and assess suitable aeration."
    )
    score += 1

if rainfall is not None and rainfall >= 50:
    risks.append("Heavy rainfall")
    actions.append(
        "Inspect silo roof, drainage and water leakage."
    )
    score += 1

if moisture is not None:
    if moisture > 14:
        risks.append("Grain moisture needs urgent review")
        actions.append(
            "Compare moisture with the crop-specific safe-storage "
            "limit; assess drying and storage conditions."
        )
        score += 3
    elif moisture >= 13:
        risks.append("Grain moisture needs verification")
        actions.append(
            "Verify the reading against the appropriate grain limit."
        )
        score += 1

if silo_temp is not None and silo_temp >= 35:
    risks.append("High internal silo temperature")
    actions.append(
        "Check for hot spots, insects and abnormal grain heating."
    )
    score += 3

# Do not call missing data a confirmed low risk
if score >= 4:
    risk_level = "HIGH"
elif score >= 2:
    risk_level = "MODERATE"
elif all(v is None for v in [temperature, humidity, rainfall,
                             moisture, silo_temp]):
    risk_level = "INSUFFICIENT DATA"
else:
    risk_level = "LOW SCREENING SCORE - MONITOR"

if not risks:
    risks.append("No warning triggered by the available measurements.")

if not actions:
    actions.append(
        "Continue routine monitoring and measure grain moisture "
        "and internal silo temperature."
    )

if moisture is None or silo_temp is None:
    actions.append(
        "For a better assessment, measure grain moisture and "
        "temperature inside the silo."
    )

# 7. Display the final report
print("\n" + "=" * 50)
print("       SMART SILO RISK REPORT")
print("=" * 50)
print("Location:", location)
print("Grain:", grain)
print("Outside temperature:", temperature, "°C")
print("Relative humidity:", humidity, "%")
print("Rainfall:", rainfall, "mm")
print("Grain moisture:", moisture, "%")
print("Internal silo temperature:", silo_temp, "°C")
print("\nSCREENING RESULT:", risk_level)

print("\nPOSSIBLE RISKS:")
for item in risks:
    print("-", item)

print("\nRECOMMENDED ACTIONS:")
for i, action in enumerate(actions, 1):
    print(f"{i}. {action}")

print("\nNote: This is a preliminary screening tool.")
print("Weather alone cannot confirm grain spoilage.")
print("Use crop-specific limits and physical inspections.")
print("=" * 50)

# 8. Save and download the report
report_file = "silo_image_analysis_report.txt"

with open(report_file, "w", encoding="utf-8") as f:
    f.write("SMART SILO RISK REPORT\n")
    f.write("=" * 40 + "\n")
    f.write(f"Location: {location}\n")
    f.write(f"Grain: {grain}\n")
    f.write(f"Temperature: {temperature} °C\n")
    f.write(f"Humidity: {humidity}%\n")
    f.write(f"Rainfall: {rainfall} mm\n")
    f.write(f"Grain moisture: {moisture}%\n")
    f.write(f"Internal silo temperature: {silo_temp} °C\n")
    f.write(f"Risk level: {risk_level}\n\n")
    f.write("Possible risks:\n")
    for item in risks:
        f.write("- " + item + "\n")
    f.write("\nRecommended actions:\n")
    for action in actions:
        f.write("- " + action + "\n")

files.download(report_file)
