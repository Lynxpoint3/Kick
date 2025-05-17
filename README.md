#include <TFT_eSPI.h>
#include <XPT2046_Touchscreen.h>
#include <Wire.h>
#include <Adafruit_MCP4728.h>
#include <vector>
#include <SD.h>
#include <SPI.h>

// Pin definitions
#define CS_PIN  7   // Touchscreen CS
#define TIRQ_PIN 2  // Touchscreen IRQ

// Trigger input pins
#define BUTTON_PIN 3  // Push button on input 3
#define TRIGGER_PIN_1 4  // Trigger for envelope 1 (PITCH)
#define TRIGGER_PIN_2 5  // Trigger for envelope 2 (AMP)
#define TRIGGER_PIN_3 6  // Trigger for envelope 3 (FILTER)
#define TRIGGER_PIN_4 8  // Trigger for envelope 4 (NOISE)
#define TRIGGER_PIN_ALL 9  // Trigger for all envelopes

// Screen dimensions
#define SCREEN_WIDTH 320
#define SCREEN_HEIGHT 240

// UI Constants
#define TOP_BAR_HEIGHT 30
#define ENV_VIEW_TOP TOP_BAR_HEIGHT
#define ENV_VIEW_WIDTH SCREEN_WIDTH
#define ENV_VIEW_HEIGHT (SCREEN_HEIGHT - TOP_BAR_HEIGHT - 40) // Reserve 40px for slider
#define NUM_TABS 6
#define TAB_WIDTH (SCREEN_WIDTH / NUM_TABS)
#define TAB_HEIGHT 30
#define MAX_POINTS 20
#define POINT_RADIUS 6
#define DOUBLE_CLICK_RADIUS 20  // Larger radius for double-click detection
#define LINE_SELECT_RADIUS 15
#define HOLD_DELAY 300
#define JITTER_BUFFER_SIZE 4
#define TIME_SLIDER_WIDTH 180
#define TIME_SLIDER_HEIGHT 20
#define TIME_SLIDER_X 70
#define TIME_SLIDER_Y (SCREEN_HEIGHT - 20) // Place slider near bottom
#define TIME_SLIDER_HANDLE_WIDTH 24        // Width of oval handle
#define TIME_SLIDER_HANDLE_HEIGHT 16       // Height of oval handle
#define TIME_MIN_VALUE 50                  // 50ms minimum time
#define TIME_MAX_VALUE 4200                // 4200ms maximum time
#define DOUBLE_CLICK_TIME 300 // Time in ms to detect a double-click

// Keyboard Constants
#define KEYBOARD_TOP (TOP_BAR_HEIGHT + 5)
#define KEYBOARD_HEIGHT (SCREEN_HEIGHT - TOP_BAR_HEIGHT - 10) // Make it fit in remaining space
#define KEY_WIDTH (SCREEN_WIDTH/10)  // Keep original width
#define KEY_HEIGHT 26  // Slightly smaller height
#define KEY_SPACING 5  // Add spacing constant for vertical gaps

// Colors
#define BG_COLOR TFT_BLACK
#define GRID_COLOR 0x3186  // Dark gray
#define POINT_COLOR TFT_WHITE
#define SELECTED_POINT_COLOR TFT_RED
#define LINE_COLOR 0x07FF  // Bright cyan
#define TEXT_COLOR TFT_WHITE
#define TAB_INACTIVE 0x3186  // Dark gray
#define TAB_ACTIVE 0x5ACB    // Brighter blue
#define SLIDER_BG 0x3186     // Dark gray
#define SLIDER_FG 0x5ACB     // Brighter blue

// UI States
enum UIState { 
  ENVELOPE_EDIT, 
  SAVE_NAME_ENTRY, 
  LOAD_FILE_SELECT 
}

void updateOutputs() {
  uint16_t dacValues[4];
  
  // Get current values from each envelope based on its triggered state
  for (int i = 0; i < 4; i++) {
    float value = envelopes[i].getCurrentValue();
    dacValues[i] = value * 4095;
  }
  
  // Send values to DAC
  dac.setChannelValue(MCP4728_CHANNEL_A, dacValues[0]);
  dac.setChannelValue(MCP4728_CHANNEL_B, dacValues[1]);
  dac.setChannelValue(MCP4728_CHANNEL_C, dacValues[2]);
  dac.setChannelValue(MCP4728_CHANNEL_D, dacValues[3]);
};

// Initialize hardware
TFT_eSPI tft = TFT_eSPI();
XPT2046_Touchscreen ts(CS_PIN, TIRQ_PIN);
Adafruit_MCP4728 dac;

// Envelope data structure
struct Point {
  int x, y;
};

struct Curve {
  int index;  // Line segment index (between points)
  int offset; // Curve offset
};

// Four envelopes (Pitch, Amp, Filter, Noise)
enum EnvelopeType { PITCH, AMP, FILTER, NOISE };
const char* ENV_NAMES[] = {"PITCH", "AMP", "FILTER", "NOISE"};
const char* ALL_TAB_NAMES[] = {"PITCH", "AMP", "FILTER", "NOISE", "SAVE", "LOAD"};
const uint16_t ENV_COLORS[] = {0x07FF, 0x07E0, 0xFD20, 0xF800}; // Cyan, Green, Yellow, Red
const uint16_t ALL_TAB_COLORS[] = {0x07FF, 0x07E0, 0xFD20, 0xF800, 0xFFFF, 0xFFFF}; // Cyan, Green, Yellow, Red, White, White

class Envelope {
public:
  std::vector<Point> points;
  std::vector<Curve> curves;
  int totalDuration = 1000;  // Duration in milliseconds
  bool triggered = false;    // Flag to indicate if this envelope is triggered
  unsigned long triggerTime = 0; // Time when envelope was triggered

  Envelope() {
    points.push_back({0, ENV_VIEW_HEIGHT * 3/4});
    points.push_back({ENV_VIEW_WIDTH - 1, ENV_VIEW_HEIGHT * 3/4});
  }

  float getValue(float time) {
    int xPos = time * ENV_VIEW_WIDTH;
    for (size_t i = 1; i < points.size(); i++) {
      if (xPos >= points[i-1].x && xPos <= points[i].x) {
        int x1 = points[i-1].x;
        int y1 = points[i-1].y;
        int x2 = points[i].x;
        int y2 = points[i].y;
        bool hasCurve = false;
        int curveOffset = 0;
        for (auto &c : curves) {
          if (c.index == static_cast<int>(i) - 1) { // Fix signedness warning
            hasCurve = true;
            curveOffset = c.offset;
            break;
          }
        }
        if (!hasCurve) {
          float t = (float)(xPos - x1) / (float)(x2 - x1);
          float yPos = y1 + t * (y2 - y1);
          return 1.0 - (yPos - ENV_VIEW_TOP) / (float)ENV_VIEW_HEIGHT;
        } else {
          int cx = (x1 + x2) / 2 + curveOffset;
          int cy = (y1 + y2) / 2;
          float t = (float)(xPos - x1) / (float)(x2 - x1);
          float u = 1.0 - t;
          float yPos = u*u*y1 + 2*u*t*cy + t*t*y2;
          return 1.0 - (yPos - ENV_VIEW_TOP) / (float)ENV_VIEW_HEIGHT;
        }
      }
    }
    return 1.0 - (points.back().y - ENV_VIEW_TOP) / (float)ENV_VIEW_HEIGHT;
  }

  // Trigger the envelope to start playing
  void trigger() {
    triggered = true;
    triggerTime = millis();
  }

  // Check if the envelope is still playing
  bool isPlaying() {
    if (!triggered) return false;
    unsigned long now = millis();
    unsigned long elapsed = now - triggerTime;
    if (elapsed >= (unsigned long)totalDuration) {
      triggered = false;
      return false;
    }
    return true;
  }

  // Get the current output value based on triggered state
  float getCurrentValue() {
    if (!triggered) return 0.0f;
    
    unsigned long now = millis();
    unsigned long elapsed = now - triggerTime;
    
    // If envelope cycle is complete, reset trigger state
    if (elapsed >= (unsigned long)totalDuration) {
      triggered = false;
      return 0.0f;
    }
    
    // Calculate phase (0.0 to 1.0) based on elapsed time
    float phase = (float)elapsed / (float)totalDuration;
    return getValue(phase);
  }
};

// Global variables
Envelope envelopes[4];
EnvelopeType currentEnvelope = PITCH;
UIState currentState = ENVELOPE_EDIT;
int selectedPoint = -1;
int selectedCurve = -1;
bool holdingPoint = false;
bool holdingCurve = false;
bool draggingTimeSlider = false;
unsigned long holdStartTime = 0;
unsigned long lastClickTime = 0;
unsigned long lastDrawTime = 0;
unsigned long lastKeyboardTouchTime = 0;
#define KEYBOARD_DEBOUNCE_TIME 300  // Minimum time between key presses in ms
int lastClickedPoint = -1;
std::vector<int> jitterBufferX;
std::vector<int> jitterBufferY;
char currentPatchName[20] = ""; // Buffer for patch name
int cursorPosition = 0;         // Cursor position for text entry
File patchDir;                  // For file browsing
int selectedFileIndex = -1;     // Currently selected file
int fileScrollOffset = 0;       // Scroll offset for file list
bool needFullRedraw = true;
#define MIN_REDRAW_INTERVAL 50  // Minimum time between redraws in ms

// Trigger input state variables
bool lastButtonState = HIGH;
bool lastTrigger1State = HIGH;
bool lastTrigger2State = HIGH;
bool lastTrigger3State = HIGH;
bool lastTrigger4State = HIGH;
bool lastTriggerAllState = HIGH;
unsigned long lastDebounceTime = 0;
#define DEBOUNCE_DELAY 50  // Debounce time in ms

// Function prototypes
void drawUI(bool fullRedraw = true);
void drawEnvelope(EnvelopeType envType);
void drawTabs(bool includeEnvelopeTabs = true);
void drawTimeSlider();
void drawGrid();
int findNearbyPoint(int x, int y, int radius = POINT_RADIUS);
int findNearbyLine(int x, int y);
void processTouch();
void updateOutputs();
void checkTriggers();
void triggerAllEnvelopes();
void removeLineOverlaps();
int stableX(int rawX);
int stableY(int rawY);
void drawKeyboard();
void processKeyboardTouch(int px, int py);
void saveCurrentPatch();
void loadSelectedPatch();
void drawFileList();
void processFileListTouch(int px, int py);
void updateEnvelopePoint(int pointIndex);
bool readLine(File &file, char* buffer, int maxLen);
void printDirectory(File dir, int numTabs);
void loadDefaultPatch();

void setup() {
  Serial.begin(115200);
  delay(1000); // Give serial time to start
  Serial.println("Starting setup...");
  
  tft.init();
  tft.setRotation(3);
  tft.fillScreen(BG_COLOR);
  
  ts.begin();
  ts.setRotation(1);
  
  // DAC initialization
  if (!dac.begin()) {
    Serial.println("Failed to find MCP4728 chip");
  }
  
  // Initialize trigger input pins
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(TRIGGER_PIN_1, INPUT_PULLUP);
  pinMode(TRIGGER_PIN_2, INPUT_PULLUP);
  pinMode(TRIGGER_PIN_3, INPUT_PULLUP);
  pinMode(TRIGGER_PIN_4, INPUT_PULLUP);
  pinMode(TRIGGER_PIN_ALL, INPUT_PULLUP);
  
  // Clear display and show SD init message
  tft.fillScreen(BG_COLOR);
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(MC_DATUM);
  tft.setTextSize(1);
  tft.drawString("INITIALIZING SD CARD...", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
  
  // Try to initialize SD with BUILTIN_SDCARD for the Teensy 4.1
  Serial.println("Initializing SD card...");
  if (!SD.begin(BUILTIN_SDCARD)) {
    Serial.println("SD Card initialization failed!");
    tft.fillScreen(BG_COLOR);
    tft.setTextColor(TFT_RED);
    tft.drawString("SD CARD ERROR", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
    delay(2000);
  } else {
    Serial.println("SD card initialized!");
    
    // Display success
    tft.fillScreen(BG_COLOR);
    tft.setTextColor(TFT_GREEN);
    tft.drawString("SD CARD OK", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
    delay(1000);
    
    // Try to create the patches directory
    Serial.println("Creating patches directory...");
    if (!SD.exists("patches")) {
      if (SD.mkdir("patches")) {
        Serial.println("Created patches directory");
      } else {
        Serial.println("Failed to create patches directory");
      }
    } else {
      Serial.println("Patches directory already exists");
    }
    
    // Test file creation
    Serial.println("Testing file write access...");
    File testFile = SD.open("test.txt", FILE_WRITE);
    if (testFile) {
      testFile.println("Test file write");
      testFile.close();
      Serial.println("Test file written successfully");
    } else {
      Serial.println("Failed to create test file");
    }
  }
  
  // Continue with normal initialization
  tft.fillScreen(BG_COLOR);
  
  // Load default patch
  loadDefaultPatch();
  
  // Draw the UI
  drawUI();
}

void loop() {
  processTouch();
  checkTriggers();
  updateOutputs();
  delay(10);
}

// Check for trigger inputs
void checkTriggers() {
  // Read current states
  bool buttonState = digitalRead(BUTTON_PIN);
  bool trigger1State = digitalRead(TRIGGER_PIN_1);
  bool trigger2State = digitalRead(TRIGGER_PIN_2);
  bool trigger3State = digitalRead(TRIGGER_PIN_3);
  bool trigger4State = digitalRead(TRIGGER_PIN_4);
  bool triggerAllState = digitalRead(TRIGGER_PIN_ALL);
  
  unsigned long currentTime = millis();
  
  // Check button (active LOW)
  if (buttonState != lastButtonState) {
    lastDebounceTime = currentTime;
  }
  
  // Process button input after debounce
  if ((currentTime - lastDebounceTime) > DEBOUNCE_DELAY) {
    // Button pressed (LOW)
    if (buttonState == LOW && lastButtonState == HIGH) {
      triggerAllEnvelopes();
      Serial.println("All envelopes triggered by button");
    }
    lastButtonState = buttonState;
  }
  
  // Check individual triggers (active LOW)
  // Trigger 1 (PITCH)
  if (trigger1State != lastTrigger1State && (currentTime - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (trigger1State == LOW && lastTrigger1State == HIGH) {
      envelopes[PITCH].trigger();
      Serial.println("PITCH envelope triggered");
    }
    lastTrigger1State = trigger1State;
  }
  
  // Trigger 2 (AMP)
  if (trigger2State != lastTrigger2State && (currentTime - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (trigger2State == LOW && lastTrigger2State == HIGH) {
      envelopes[AMP].trigger();
      Serial.println("AMP envelope triggered");
    }
    lastTrigger2State = trigger2State;
  }
  
  // Trigger 3 (FILTER)
  if (trigger3State != lastTrigger3State && (currentTime - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (trigger3State == LOW && lastTrigger3State == HIGH) {
      envelopes[FILTER].trigger();
      Serial.println("FILTER envelope triggered");
    }
    lastTrigger3State = trigger3State;
  }
  
  // Trigger 4 (NOISE)
  if (trigger4State != lastTrigger4State && (currentTime - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (trigger4State == LOW && lastTrigger4State == HIGH) {
      envelopes[NOISE].trigger();
      Serial.println("NOISE envelope triggered");
    }
    lastTrigger4State = trigger4State;
  }
  
  // Trigger ALL
  if (triggerAllState != lastTriggerAllState && (currentTime - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (triggerAllState == LOW && lastTriggerAllState == HIGH) {
      triggerAllEnvelopes();
      Serial.println("All envelopes triggered by input");
    }
    lastTriggerAllState = triggerAllState;
  }
}

// Trigger all envelopes at once
void triggerAllEnvelopes() {
  for (int i = 0; i < 4; i++) {
    envelopes[i].trigger();
  }
}

void drawUI(bool fullRedraw) {
  if (currentState == ENVELOPE_EDIT) {
    if (fullRedraw) {
      tft.fillScreen(BG_COLOR);
      drawGrid();
      drawTabs();
      drawTimeSlider();
    }
    drawEnvelope(currentEnvelope);
  }
}

void drawGrid() {
  for (int y = ENV_VIEW_TOP; y <= ENV_VIEW_TOP + ENV_VIEW_HEIGHT; y += ENV_VIEW_HEIGHT / 4) {
    tft.drawLine(0, y, SCREEN_WIDTH, y, GRID_COLOR);
  }
  for (int x = 0; x <= SCREEN_WIDTH; x += SCREEN_WIDTH / 8) {
    tft.drawLine(x, ENV_VIEW_TOP, x, ENV_VIEW_TOP + ENV_VIEW_HEIGHT, GRID_COLOR);
  }
  tft.drawRect(0, ENV_VIEW_TOP, SCREEN_WIDTH, ENV_VIEW_HEIGHT, GRID_COLOR);
}

void drawEnvelope(EnvelopeType envType) {
  Envelope &env = envelopes[envType];
  uint16_t envColor = ENV_COLORS[envType];
  
  // Draw envelope lines and curves
  for (size_t i = 1; i < env.points.size(); i++) {
    Point p1 = env.points[i-1];
    Point p2 = env.points[i];
    bool hasCurve = false;
    int curveOffset = 0;
    for (auto &c : env.curves) {
      if (c.index == static_cast<int>(i) - 1) { // Fix signedness warning
        hasCurve = true;
        curveOffset = c.offset;
        break;
      }
    }
    if (!hasCurve) {
      tft.drawLine(p1.x, p1.y, p2.x, p2.y, envColor);
    } else {
      int xDist = p2.x - p1.x;
      int maxOffset = xDist / 2;
      curveOffset = constrain(curveOffset, -maxOffset, maxOffset);
      for (auto &c : env.curves) {
        if (c.index == static_cast<int>(i) - 1) { // Fix signedness warning
          c.offset = curveOffset;
          break;
        }
      }
      
      // Calculate control point based on curve offset
      int cx = (p1.x + p2.x) / 2 + curveOffset;
      int cy = (p1.y + p2.y) / 2;
      
      // Draw a smooth curve using more segments for better quality
      for (int t = 0; t <= 30; t++) {
        float u = t / 30.0;
        float v = (t+1) / 30.0;
        int x1 = (1-u)*(1-u)*p1.x + 2*(1-u)*u*cx + u*u*p2.x;
        int y1 = (1-u)*(1-u)*p1.y + 2*(1-u)*u*cy + u*u*p2.y;
        int x2 = (1-v)*(1-v)*p1.x + 2*(1-v)*v*cx + v*v*p2.x; 
        int y2 = (1-v)*(1-v)*p1.y + 2*(1-v)*v*cy + v*v*p2.y;
        tft.drawLine(x1, y1, x2, y2, envColor);
      }
    }
  }
  
  // Draw points
  for (size_t i = 0; i < env.points.size(); i++) {
    if (static_cast<int>(i) == selectedPoint && holdingPoint) { // Fix signedness warning
      tft.fillCircle(env.points[i].x, env.points[i].y, POINT_RADIUS, SELECTED_POINT_COLOR);
    } else {
      tft.fillCircle(env.points[i].x, env.points[i].y, POINT_RADIUS, POINT_COLOR);
      tft.drawCircle(env.points[i].x, env.points[i].y, POINT_RADIUS, envColor);
    }
  }
}

void drawTabs(bool includeEnvelopeTabs) {
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(MC_DATUM);
  tft.setTextSize(1);
  
  int startTab = 0;
  int endTab = NUM_TABS;
  
  if (!includeEnvelopeTabs) {
    startTab = 4; // Only draw SAVE and LOAD tabs
  }
  
  for (int i = startTab; i < endTab; i++) {
    int x = i * TAB_WIDTH;
    int y = TOP_BAR_HEIGHT / 2;
    uint16_t tabColor = TAB_INACTIVE;
    
    if (currentState == ENVELOPE_EDIT && i == currentEnvelope) {
      tabColor = TAB_ACTIVE;
    } else if (currentState == SAVE_NAME_ENTRY && i == 4) { // SAVE tab
      tabColor = TAB_ACTIVE;
    } else if (currentState == LOAD_FILE_SELECT && i == 5) { // LOAD tab
      tabColor = TAB_ACTIVE;
    }
    
    uint16_t lineColor = i < 4 ? ENV_COLORS[i] : ALL_TAB_COLORS[i];
    tft.fillRoundRect(x + 2, 2, TAB_WIDTH - 4, TOP_BAR_HEIGHT - 4, 4, tabColor);
    
    if (i == currentEnvelope && currentState == ENVELOPE_EDIT) {
      tft.drawLine(x + 2, TOP_BAR_HEIGHT - 2, x + TAB_WIDTH - 2, TOP_BAR_HEIGHT - 2, lineColor);
    }
    
    tft.drawString(ALL_TAB_NAMES[i], x + TAB_WIDTH/2, y);
  }
}

void drawTimeSlider() {
  Envelope &env = envelopes[currentEnvelope];
  
  // Calculate slider position (where the handle should be)
  int sliderPos = map(env.totalDuration, TIME_MIN_VALUE, TIME_MAX_VALUE, 0, TIME_SLIDER_WIDTH);
  int handleX = TIME_SLIDER_X + sliderPos;
  
  // Clear the slider area first
  tft.fillRect(TIME_SLIDER_X - TIME_SLIDER_HANDLE_WIDTH/2, 
               TIME_SLIDER_Y - TIME_SLIDER_HANDLE_HEIGHT/2 - 5,
               TIME_SLIDER_WIDTH + TIME_SLIDER_HANDLE_WIDTH, 
               TIME_SLIDER_HANDLE_HEIGHT + 10, 
               BG_COLOR);
  
  // Draw slider track (line)
  tft.drawLine(TIME_SLIDER_X, TIME_SLIDER_Y, 
               TIME_SLIDER_X + TIME_SLIDER_WIDTH, TIME_SLIDER_Y, 
               GRID_COLOR);
              
  // Draw small tick marks
  for (int i = 0; i <= 4; i++) {
    int x = TIME_SLIDER_X + (TIME_SLIDER_WIDTH * i / 4);
    tft.drawLine(x, TIME_SLIDER_Y - 3, x, TIME_SLIDER_Y + 3, GRID_COLOR);
  }
  
  // Draw oval handle with value inside
  tft.fillRoundRect(handleX - TIME_SLIDER_HANDLE_WIDTH/2, 
                   TIME_SLIDER_Y - TIME_SLIDER_HANDLE_HEIGHT/2,
                   TIME_SLIDER_HANDLE_WIDTH, 
                   TIME_SLIDER_HANDLE_HEIGHT, 
                   TIME_SLIDER_HANDLE_HEIGHT/2, 
                   SLIDER_FG);
                   
  tft.drawRoundRect(handleX - TIME_SLIDER_HANDLE_WIDTH/2, 
                   TIME_SLIDER_Y - TIME_SLIDER_HANDLE_HEIGHT/2,
                   TIME_SLIDER_HANDLE_WIDTH, 
                   TIME_SLIDER_HANDLE_HEIGHT, 
                   TIME_SLIDER_HANDLE_HEIGHT/2, 
                   TFT_WHITE);
  
  // Format time value
  char timeText[8];
  if (env.totalDuration >= 1000) {
    float seconds = env.totalDuration / 1000.0;
    sprintf(timeText, "%.1fs", seconds);
  } else {
    sprintf(timeText, "%dms", env.totalDuration);
  }
  
  // Draw time text inside the oval
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(MC_DATUM);
  tft.setTextSize(1);
  tft.drawString(timeText, handleX, TIME_SLIDER_Y);
}

int findNearbyPoint(int x, int y, int radius) {
  Envelope &env = envelopes[currentEnvelope];
  int closestPoint = -1;
  int minDistSq = radius * radius + 1; // Initialize with value just outside radius
  
  for (size_t i = 0; i < env.points.size(); i++) {
    int dx = env.points[i].x - x;
    int dy = env.points[i].y - y;
    int distSq = dx*dx + dy*dy;
    if (distSq <= radius*radius && distSq < minDistSq) {
      minDistSq = distSq;
      closestPoint = i;
    }
  }
  return closestPoint;
}
int findNearbyLine(int x, int y) {
  Envelope &env = envelopes[currentEnvelope];
  for (size_t i = 1; i < env.points.size(); i++) {
    int x1 = env.points[i-1].x;
    int y1 = env.points[i-1].y;
    int x2 = env.points[i].x;
    int y2 = env.points[i].y;
    if (x >= min(x1, x2) - LINE_SELECT_RADIUS && x <= max(x1, x2) + LINE_SELECT_RADIUS) {
      float dx = x2 - x1;
      float dy = y2 - y1;
      float lenSq = dx*dx + dy*dy;
      if (lenSq == 0) continue;
      float t = max(0.0f, min(1.0f, ((x - x1) * dx + (y - y1) * dy) / lenSq));
      float projX = x1 + t * dx;
      float projY = y1 + t * dy;
      float distSq = (x - projX)*(x - projX) + (y - projY)*(y - projY);
      if (distSq <= LINE_SELECT_RADIUS*LINE_SELECT_RADIUS) {
        return i-1;
      }
    }
  }
  return -1;
}

bool isInTimeSlider(int x, int y) {
  // Calculate current slider handle position
  int sliderPos = map(envelopes[currentEnvelope].totalDuration, 
                     TIME_MIN_VALUE, TIME_MAX_VALUE, 
                     0, TIME_SLIDER_WIDTH);
  int handleX = TIME_SLIDER_X + sliderPos;
  
  // Check if point is inside the oval-shaped handle
  return (x >= handleX - TIME_SLIDER_HANDLE_WIDTH/2 &&
          x <= handleX + TIME_SLIDER_HANDLE_WIDTH/2 &&
          y >= TIME_SLIDER_Y - TIME_SLIDER_HANDLE_HEIGHT/2 &&
          y <= TIME_SLIDER_Y + TIME_SLIDER_HANDLE_HEIGHT/2);
}

int getEnvelopeTabAt(int x, int y) {
  if (y <= TOP_BAR_HEIGHT) {
    for (int i = 0; i < NUM_TABS; i++) {
      if (x >= i * TAB_WIDTH && x < (i+1) * TAB_WIDTH) {
        return i;
      }
    }
  }
  return -1;
}

void removeLineOverlaps() {
  Envelope &env = envelopes[currentEnvelope];
  std::sort(env.points.begin(), env.points.end(), [](Point a, Point b) {
    return a.x < b.x;
  });
  for (size_t i = 1; i < env.points.size(); ) {
    if (env.points[i].x == env.points[i-1].x) {
      env.points.erase(env.points.begin() + i);
    } else {
      ++i;
    }
  }
  for (size_t i = 0; i < env.curves.size(); ) {
    if (env.curves[i].index >= static_cast<int>(env.points.size()) - 1) { // Fix signedness warning
      env.curves.erase(env.curves.begin() + i);
    } else {
      ++i;
    }
  }void loadDefaultPatch() {
  // Path to default patch file
  const char* defaultPatchPath = "patches/DEFAULT.kck";
  
  // Check if the default patch exists
  if (SD.exists(defaultPatchPath)) {
    Serial.println("Found DEFAULT.kck, loading it...");
    File patchFile = SD.open(defaultPatchPath);
    
    if (patchFile) {
      // Read header - line by line processing
      char buffer[50];
      if (readLine(patchFile, buffer, sizeof(buffer)) && 
          strcmp(buffer, "KICK_PATCH") == 0) {
        // Skip version line
        readLine(patchFile, buffer, sizeof(buffer));
        
        // Read envelopes
        while (patchFile.available()) {
          if (!readLine(patchFile, buffer, sizeof(buffer))) {
            break;
          }
          
          // Check for envelope section
          if (strncmp(buffer, "ENVELOPE:", 9) == 0) {
            int envIndex = atoi(buffer + 9);
            
            // Read duration
            if (readLine(patchFile, buffer, sizeof(buffer)) && 
                strncmp(buffer, "DURATION:", 9) == 0) {
              int duration = atoi(buffer + 9);
              // Make sure the loaded duration is within our valid range
              duration = constrain(duration, TIME_MIN_VALUE, TIME_MAX_VALUE);
              envelopes[envIndex].totalDuration = duration;
              Serial.print("Loaded duration for env ");
              Serial.print(envIndex);
              Serial.print(": ");
              Serial.println(duration);
            }
            
            // Read points
            if (readLine(patchFile, buffer, sizeof(buffer)) && 
                strncmp(buffer, "POINTS:", 7) == 0) {
              int numPoints = atoi(buffer + 7);
              envelopes[envIndex].points.clear();
              
              for (int i = 0; i < numPoints; i++) {
                if (readLine(patchFile, buffer, sizeof(buffer))) {
                  char* comma = strchr(buffer, ',');
                  if (comma) {
                    *comma = 0; // Split string at comma
                    int x = atoi(buffer);
                    int y = atoi(comma + 1);
                    envelopes[envIndex].points.push_back({x, y});
                  }
                }
              }
              
              // Read curves
              if (readLine(patchFile, buffer, sizeof(buffer)) && 
                  strncmp(buffer, "CURVES:", 7) == 0) {
                int numCurves = atoi(buffer + 7);
                envelopes[envIndex].curves.clear();
                
                for (int i = 0; i < numCurves; i++) {
                  if (readLine(patchFile, buffer, sizeof(buffer))) {
                    char* comma = strchr(buffer, ',');
                    if (comma) {
                      *comma = 0; // Split string at comma
                      int index = atoi(buffer);
                      int offset = atoi(comma + 1);
                      envelopes[envIndex].curves.push_back({index, offset});
                    }
                  }
                }
              }
            }
          }
        }
        
        Serial.println("Default patch loaded successfully");
      }
      
      patchFile.close();
    }
  } else {
    Serial.println("Default patch not found");
  }
}

void processTouch() {
  if (ts.touched()) {
    TS_Point p = ts.getPoint();
    int px = map(p.x, 3900, 300, -5, 315);
    int py = map(p.y, 3900, 300, 0, 240);
    px = constrain(stableX(px), 0, SCREEN_WIDTH);
    py = constrain(stableY(py), 0, SCREEN_HEIGHT);
    
    // Handle different UI states
    if (currentState == SAVE_NAME_ENTRY) {
      processKeyboardTouch(px, py);
      return;
    } else if (currentState == LOAD_FILE_SELECT) {
      processFileListTouch(px, py);
      return;
    }
    
    // Handle top tab selection first
    int tabIndex = getEnvelopeTabAt(px, py);
    if (tabIndex >= 0) {
      if (tabIndex < 4) {
        // Envelope tabs
        currentEnvelope = (EnvelopeType)tabIndex;
        currentState = ENVELOPE_EDIT;
        selectedPoint = -1;
        selectedCurve = -1;
        holdingPoint = false;
        holdingCurve = false;
        lastClickedPoint = -1;
        drawUI();
      } else if (tabIndex == 4) {
        // Save tab
        if (currentState == SAVE_NAME_ENTRY) {
          // If already in save menu, close it
          currentState = ENVELOPE_EDIT;
          drawUI();
        } else {
          // Enter save menu
          currentState = SAVE_NAME_ENTRY;
          cursorPosition = 0;
          currentPatchName[0] = '\0';
          tft.fillScreen(BG_COLOR);
          drawTabs(false); // Draw only save/load tabs
          drawKeyboard();
        }
      } else if (tabIndex == 5) {
        // Load tab
        if (currentState == LOAD_FILE_SELECT) {
          // If already in load menu, close it
          currentState = ENVELOPE_EDIT;
          drawUI();
        } else {
          // Enter load menu
          currentState = LOAD_FILE_SELECT;
          selectedFileIndex = -1;
          fileScrollOffset = 0;
          tft.fillScreen(BG_COLOR);
          drawTabs(false); // Draw only save/load tabs
          drawFileList();
        }
      }
      return;
    }
    
    // Handle time slider interaction
    if (currentState == ENVELOPE_EDIT) {
      // Check for time slider interaction first, before any other interactions
      if (draggingTimeSlider || isInTimeSlider(px, py)) {
        draggingTimeSlider = true;
        // Update position
        int newPos = constrain(px - TIME_SLIDER_X, 0, TIME_SLIDER_WIDTH);
        int newTime = map(newPos, 0, TIME_SLIDER_WIDTH, TIME_MIN_VALUE, TIME_MAX_VALUE);
        envelopes[currentEnvelope].totalDuration = newTime;
        drawTimeSlider();
        return;  // Important: stop processing other touches
      }
      
      // Only if we're not touching the slider, proceed with point/curve handling
      if (py >= ENV_VIEW_TOP && py <= ENV_VIEW_TOP + ENV_VIEW_HEIGHT) {
        Envelope &env = envelopes[currentEnvelope];
        unsigned long now = millis();
        
        if (holdingPoint) {
          env.points[selectedPoint].x = px;
          env.points[selectedPoint].y = py;
          env.points[selectedPoint].x = constrain(env.points[selectedPoint].x, 0, SCREEN_WIDTH);
          env.points[selectedPoint].y = constrain(env.points[selectedPoint].y, ENV_VIEW_TOP, ENV_VIEW_TOP + ENV_VIEW_HEIGHT);
          if (selectedPoint == 0) {
            env.points[selectedPoint].x = 0;
          } else if (selectedPoint == static_cast<int>(env.points.size()) - 1) {
            env.points[selectedPoint].x = SCREEN_WIDTH - 1;
          }
          removeLineOverlaps(); // Handles drag-over deletion
          
          // Throttle redraw for better performance
          unsigned long now = millis();
          if (now - lastDrawTime >= MIN_REDRAW_INTERVAL) {
            lastDrawTime = now;
            drawUI();
          }
        } else if (holdingCurve) {
          for (auto &c : env.curves) {
            if (c.index == selectedCurve) {
              // Get the line's start and end points
              int x1 = env.points[c.index].x;
              int y1 = env.points[c.index].y;
              int x2 = env.points[c.index + 1].x;
              int y2 = env.points[c.index + 1].y;
              
              // Calculate the perpendicular distance from cursor to line
              float dx = x2 - x1;
              float dy = y2 - y1;
              
              // Project cursor onto the line
              float t = constrain(((px - x1) * dx + (py - y1) * dy) / (dx*dx + dy*dy), 0.0f, 1.0f);
              float projX = x1 + t * dx;
              float projY = y1 + t * dy;
              
              // Calculate perpendicular vector
              float perpX = px - projX;
              float perpY = py - projY;
              float perpLength = sqrt(perpX*perpX + perpY*perpY);
              
              // Calculate offset based on perpendicular distance and projection point
              if (perpLength > 0) {
                // Set offset based on cursor position relative to the line
                // We want the peak of the curve to follow the cursor
                int midPointX = (x1 + x2) / 2;
                float offsetScale = 2.0f * (t - 0.5f);
                c.offset = round(perpLength * (perpX / perpLength)) - ((projX - midPointX) * offsetScale);
              }
              
              // Constrain offset to prevent extreme curves
              int xDist = x2 - x1;
              int maxOffset = xDist / 2;
              c.offset = constrain(c.offset, -maxOffset, maxOffset);
              
              // Throttle redraw for better performance
              if (now - lastDrawTime >= MIN_REDRAW_INTERVAL) {
                lastDrawTime = now;
                drawUI();
              }
              break;
            }
          }
        } else {
          // Use larger radius for all point interactions
          int pointIndex = findNearbyPoint(px, py, DOUBLE_CLICK_RADIUS);
          if (pointIndex >= 0) {
            // Double-click detection
            if (pointIndex == lastClickedPoint && (now - lastClickTime) <= DOUBLE_CLICK_TIME) {
              // Ensure we don't delete if there are only 2 points (minimum required)
              if (env.points.size() > 2) {
                env.points.erase(env.points.begin() + pointIndex);
                lastClickedPoint = -1;
                drawUI();
              }
              return;
            }
            
            lastClickedPoint = pointIndex;
            lastClickTime = now;
            
            selectedPoint = pointIndex;
            holdingPoint = true;
            holdStartTime = now;
            drawUI();
            return;
          }
          
          int lineIndex = findNearbyLine(px, py);
          if (lineIndex >= 0) {
            selectedCurve = lineIndex;
            bool curveExists = false;
            for (auto &c : env.curves) {
              if (c.index == lineIndex) {
                curveExists = true;
                break;
              }
            }
            if (!curveExists) {
              env.curves.push_back({lineIndex, 0});
            }
            holdingCurve = true;
            holdStartTime = now;
            drawUI();
            return;
          }
          
          // Add a point when clicking in empty space (not on a line or point)
          if (env.points.size() < MAX_POINTS) {
            bool tooClose = false;
            for (auto &pt : env.points) {
              if (abs(pt.x - px) < POINT_RADIUS) {
                tooClose = true;
                break;
              }
            }
            if (!tooClose) {
              auto it = env.points.begin();
              while (it != env.points.end() && it->x < px) ++it;
              env.points.insert(it, {px, py});
              removeLineOverlaps();
              drawUI();
            }
          }
        }
      }
    }
  } else {
    holdingPoint = false;
    holdingCurve = false;
    draggingTimeSlider = false;
    selectedPoint = -1;
    selectedCurve = -1;
    jitterBufferX.clear();
    jitterBufferY.clear();
  }
}
