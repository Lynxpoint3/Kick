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
#define TIME_MIN_VALUE 50    // 50ms
#define TIME_MAX_VALUE 3500  // 3500ms
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



void drawUI(bool fullRedraw = true);
void drawEnvelope(EnvelopeType envType);
void drawTabs(bool includeEnvelopeTabs = true);
void drawTimeSlider();
void drawGrid();
int findNearbyPoint(int x, int y, int radius = POINT_RADIUS);
int findNearbyLine(int x, int y);
void processTouch();
void updateOutputs();
void removeLineOverlaps();
int stableX(int rawX);
int stableY(int rawY);
void drawKeyboard();
void processKeyboardTouch(int px, int py);
void saveCurrentPatch();
void loadSelectedPatch();
void loadDefaultPatch();
void drawFileList();
void processFileListTouch(int px, int py);
void updateEnvelopePoint(int pointIndex);
bool readLine(File &file, char* buffer, int maxLen);
void printDirectory(File dir, int numTabs);

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
  
  // Clear display and show SD init message
  tft.fillScreen(BG_COLOR);
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(MC_DATUM);
  tft.setTextSize(1);
  tft.drawString("INITIALIZING SD CARD...", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
  
  // Try to initialize SD without any parameters (for built-in SD slot)
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
  loadDefaultPatch();
  drawUI();
}

void loop() {
  processTouch();
  updateOutputs();
  delay(10);
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

int findNearbyPoint(int x, int y, int radius = POINT_RADIUS) {
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
  }
}

void updateOutputs() {
  float values[4];
  for (int i = 0; i < 4; i++) {
    unsigned long now = millis();
    float phase = (now % envelopes[i].totalDuration) / (float)envelopes[i].totalDuration;
    values[i] = envelopes[i].getValue(phase);
  }
  uint16_t dacValues[4];
  for (int i = 0; i < 4; i++) {
    dacValues[i] = values[i] * 4095;
  }
  dac.setChannelValue(MCP4728_CHANNEL_A, dacValues[0]);
  dac.setChannelValue(MCP4728_CHANNEL_B, dacValues[1]);
  dac.setChannelValue(MCP4728_CHANNEL_C, dacValues[2]);
  dac.setChannelValue(MCP4728_CHANNEL_D, dacValues[3]);
}

int stableX(int rawX) {
  jitterBufferX.push_back(rawX);
  if (jitterBufferX.size() > JITTER_BUFFER_SIZE) jitterBufferX.erase(jitterBufferX.begin());
  int sum = 0;
  for (int val : jitterBufferX) sum += val;
  return sum / jitterBufferX.size();
}

int stableY(int rawY) {
  jitterBufferY.push_back(rawY);
  if (jitterBufferY.size() > JITTER_BUFFER_SIZE) jitterBufferY.erase(jitterBufferY.begin());
  int sum = 0;
  for (int val : jitterBufferY) sum += val;
  return sum / jitterBufferY.size();
}

void updateEnvelopePoint(int pointIndex) {
  // Only redraw if enough time has passed since last redraw
  unsigned long now = millis();
  if (now - lastDrawTime < MIN_REDRAW_INTERVAL) return;
  lastDrawTime = now;
  
  Envelope &env = envelopes[currentEnvelope];
  uint16_t envColor = ENV_COLORS[currentEnvelope];
  
  // Erase and redraw lines connected to this point
  if (pointIndex > 0) {
    // Erase and redraw the line from the previous point
    Point prev = env.points[pointIndex-1];
    Point curr = env.points[pointIndex];
    
    // First erase by drawing with background color (simple approach)
    int x1 = max(0, min(prev.x, curr.x) - POINT_RADIUS * 2);
    int y1 = max(ENV_VIEW_TOP, min(prev.y, curr.y) - POINT_RADIUS * 2);
    int x2 = min(SCREEN_WIDTH, max(prev.x, curr.x) + POINT_RADIUS * 2);
    int y2 = min(ENV_VIEW_TOP + ENV_VIEW_HEIGHT, max(prev.y, curr.y) + POINT_RADIUS * 2);
    
    tft.fillRect(x1, y1, x2-x1, y2-y1, BG_COLOR);
    
    // Then redraw grid in that area
    // (simplified - for a full solution you'd redraw grid lines more precisely)
    
    // Then redraw the line
    bool hasCurve = false;
    for (auto &c : env.curves) {
      if (c.index == pointIndex - 1) {
        hasCurve = true;
        // Draw curve code here (omitted for brevity)
        break;
      }
    }
    
    if (!hasCurve) {
      tft.drawLine(prev.x, prev.y, curr.x, curr.y, envColor);
    }
  }
  
  if (pointIndex < static_cast<int>(env.points.size()) - 1) {
    // Similar code for the next line
    // Omitted for brevity
  }
  
  // Redraw the point itself
  tft.fillCircle(env.points[pointIndex].x, env.points[pointIndex].y, POINT_RADIUS, SELECTED_POINT_COLOR);
  tft.drawCircle(env.points[pointIndex].x, env.points[pointIndex].y, POINT_RADIUS, envColor);
}

void drawKeyboard() {
  // Draw keyboard background
  tft.fillRect(0, KEYBOARD_TOP, SCREEN_WIDTH, KEYBOARD_HEIGHT, GRID_COLOR);
  tft.drawRect(0, KEYBOARD_TOP, SCREEN_WIDTH, KEYBOARD_HEIGHT, 0xFFFF);
  
  // Draw current filename entry box
  tft.fillRect(10, KEYBOARD_TOP + 10, SCREEN_WIDTH - 20, 25, BG_COLOR);
  tft.drawRect(10, KEYBOARD_TOP + 10, SCREEN_WIDTH - 20, 25, 0xFFFF);
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(ML_DATUM);
  tft.setTextSize(2);
  tft.drawString(currentPatchName, 15, KEYBOARD_TOP + 25);
  
  // Draw cursor
  int cursorX = 15;
  for (int i = 0; i < cursorPosition; i++) {
    cursorX += tft.textWidth(currentPatchName[i]);
  }
  tft.drawFastVLine(cursorX, KEYBOARD_TOP + 15, 20, 0xFFFF);
  
  // Calculate vertical positions with even spacing
  int startY = KEYBOARD_TOP + 45;
  int row1Y = startY;
  int row2Y = startY + KEY_HEIGHT + KEY_SPACING;
  int row3Y = row2Y + KEY_HEIGHT + KEY_SPACING;
  int row4Y = row3Y + KEY_HEIGHT + KEY_SPACING;
  int row5Y = row4Y + KEY_HEIGHT + KEY_SPACING;
  
  // Keys row 1 - numbers
  tft.setTextSize(2);  // Larger text (was 1)
  for (int i = 0; i < 10; i++) {
    tft.fillRoundRect(i * KEY_WIDTH + 2, row1Y, KEY_WIDTH - 4, KEY_HEIGHT, 4, 0x8410);
    tft.setTextDatum(MC_DATUM);
    tft.drawString(String(i), i * KEY_WIDTH + KEY_WIDTH/2, row1Y + KEY_HEIGHT/2);
  }
  
  // Keys row 2 - QWERTYUIOP
  const char* row1_keys = "QWERTYUIOP";
  for (int i = 0; i < 10; i++) {
    tft.fillRoundRect(i * KEY_WIDTH + 2, row2Y, KEY_WIDTH - 4, KEY_HEIGHT, 4, 0x8410);
    tft.drawString(String(row1_keys[i]), i * KEY_WIDTH + KEY_WIDTH/2, row2Y + KEY_HEIGHT/2);
  }
  
  // Keys row 3 - ASDFGHJKL
  const char* row2_keys = "ASDFGHJKL";
  for (int i = 0; i < 9; i++) {
    tft.fillRoundRect((i + 0.5) * KEY_WIDTH + 2, row3Y, KEY_WIDTH - 4, KEY_HEIGHT, 4, 0x8410);
    tft.drawString(String(row2_keys[i]), (i + 0.5) * KEY_WIDTH + KEY_WIDTH/2, row3Y + KEY_HEIGHT/2);
  }
  
  // Keys row 4 - ZXCVBNM
  const char* row3_keys = "ZXCVBNM";
  for (int i = 0; i < 7; i++) {
    tft.fillRoundRect((i + 1.5) * KEY_WIDTH + 2, row4Y, KEY_WIDTH - 4, KEY_HEIGHT, 4, 0x8410);
    tft.drawString(String(row3_keys[i]), (i + 1.5) * KEY_WIDTH + KEY_WIDTH/2, row4Y + KEY_HEIGHT/2);
  }
  
  // Special keys
  tft.fillRoundRect(1 * KEY_WIDTH + 2, row5Y, 3 * KEY_WIDTH - 4, KEY_HEIGHT, 4, 0x8410);
  tft.drawString("SPACE", 2.5 * KEY_WIDTH, row5Y + KEY_HEIGHT/2);
  
  tft.fillRoundRect(4.5 * KEY_WIDTH + 2, row5Y, 2 * KEY_WIDTH - 4, KEY_HEIGHT, 4, 0x8410);
  tft.drawString("CLEAR", 5.5 * KEY_WIDTH, row5Y + KEY_HEIGHT/2);
  
  tft.fillRoundRect(7 * KEY_WIDTH + 2, row5Y, 2 * KEY_WIDTH - 4, KEY_HEIGHT, 4, 0x04A0);
  tft.drawString("SAVE", 8 * KEY_WIDTH, row5Y + KEY_HEIGHT/2);
}

void processKeyboardTouch(int px, int py) {
  // Check if touch is in the keyboard area
  if (py < KEYBOARD_TOP) return;
  
  // Calculate vertical positions with even spacing
  int startY = KEYBOARD_TOP + 45;
  int row1Y = startY;
  int row2Y = startY + KEY_HEIGHT + KEY_SPACING;
  int row3Y = row2Y + KEY_HEIGHT + KEY_SPACING;
  int row4Y = row3Y + KEY_HEIGHT + KEY_SPACING;
  int row5Y = row4Y + KEY_HEIGHT + KEY_SPACING;
  
  // Check for key presses - row 1 (numbers)
  if (py >= row1Y && py < row1Y + KEY_HEIGHT) {
    int keyIndex = px / KEY_WIDTH;
    if (keyIndex >= 0 && keyIndex < 10 && cursorPosition < 19) {
      currentPatchName[cursorPosition++] = '0' + keyIndex;
      currentPatchName[cursorPosition] = '\0';
      drawKeyboard();
    }
  }
  // Row 2 - QWERTYUIOP
  else if (py >= row2Y && py < row2Y + KEY_HEIGHT) {
    int keyIndex = px / KEY_WIDTH;
    if (keyIndex >= 0 && keyIndex < 10 && cursorPosition < 19) {
      currentPatchName[cursorPosition++] = "QWERTYUIOP"[keyIndex];
      currentPatchName[cursorPosition] = '\0';
      drawKeyboard();
    }
  }
  // Row 3 - ASDFGHJKL
  else if (py >= row3Y && py < row3Y + KEY_HEIGHT) {
    int keyIndex = (px - KEY_WIDTH/2) / KEY_WIDTH;
    if (keyIndex >= 0 && keyIndex < 9 && cursorPosition < 19) {
      currentPatchName[cursorPosition++] = "ASDFGHJKL"[keyIndex];
      currentPatchName[cursorPosition] = '\0';
      drawKeyboard();
    }
  }
  // Row 4 - ZXCVBNM
  else if (py >= row4Y && py < row4Y + KEY_HEIGHT) {
    int keyIndex = (px - KEY_WIDTH*1.5) / KEY_WIDTH;
    if (keyIndex >= 0 && keyIndex < 7 && cursorPosition < 19) {
      currentPatchName[cursorPosition++] = "ZXCVBNM"[keyIndex];
      currentPatchName[cursorPosition] = '\0';
      drawKeyboard();
    }
  }
  // Special keys
  else if (py >= row5Y && py < row5Y + KEY_HEIGHT) {
    // Space key
    if (px >= 1 * KEY_WIDTH && px < 4 * KEY_WIDTH && cursorPosition < 19) {
      currentPatchName[cursorPosition++] = ' ';
      currentPatchName[cursorPosition] = '\0';
      drawKeyboard();
    }
    // Clear key
    else if (px >= 4.5 * KEY_WIDTH && px < 6.5 * KEY_WIDTH) {
      cursorPosition = 0;
      currentPatchName[0] = '\0';
      drawKeyboard();
    }
    // Save key
    else if (px >= 7 * KEY_WIDTH && px < 9 * KEY_WIDTH) {
      if (strlen(currentPatchName) > 0) {
        saveCurrentPatch();
        currentState = ENVELOPE_EDIT;
        drawUI();
      }
    }
  }
}

void drawFileList() {
  Serial.println("Drawing file list");
  
  // Draw file list background
  tft.fillRect(0, TOP_BAR_HEIGHT, SCREEN_WIDTH, SCREEN_HEIGHT - TOP_BAR_HEIGHT, BG_COLOR);
  tft.drawRect(0, TOP_BAR_HEIGHT, SCREEN_WIDTH, SCREEN_HEIGHT - TOP_BAR_HEIGHT, GRID_COLOR);
  
  // Draw instructions
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(MC_DATUM);
  tft.setTextSize(1);
  tft.drawString("SELECT A PATCH TO LOAD", SCREEN_WIDTH/2, TOP_BAR_HEIGHT + 15);
  
  // Make sure patches directory exists
  if (!SD.exists("patches")) {
    Serial.println("Patches directory doesn't exist");
    SD.mkdir("patches");
    tft.setTextColor(TEXT_COLOR);
    tft.setTextDatum(MC_DATUM);
    tft.drawString("NO PATCHES FOUND", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
    return;
  }
  
  // Open patches directory
  Serial.println("Opening patches directory");
  File patchDir = SD.open("patches");
  
  if (!patchDir) {
    Serial.println("Failed to open patches directory");
    tft.setTextColor(TEXT_COLOR);
    tft.setTextDatum(MC_DATUM);
    tft.drawString("ERROR OPENING PATCHES", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
    return;
  }
  
  if (!patchDir.isDirectory()) {
    Serial.println("Patches is not a directory");
    patchDir.close();
    tft.setTextColor(TEXT_COLOR);
    tft.setTextDatum(MC_DATUM);
    tft.drawString("PATCHES IS NOT A DIRECTORY", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
    return;
  }
  
  // Count patches
  Serial.println("Counting patches");
  int patchCount = 0;
  File entry;
  
  while (true) {
    entry = patchDir.openNextFile();
    if (!entry) {
      break;
    }
    
    const char* entryName = entry.name();
    Serial.print("Found file: ");
    Serial.println(entryName);
    
    int nameLen = strlen(entryName);
    if (nameLen > 4 && strcmp(entryName + nameLen - 4, ".kck") == 0) {
      patchCount++;
    }
    entry.close();
  }
  
  Serial.print("Found ");
  Serial.print(patchCount);
  Serial.println(" patches");
  
  if (patchCount == 0) {
    patchDir.close();
    tft.setTextColor(TEXT_COLOR);
    tft.setTextDatum(MC_DATUM);
    tft.drawString("NO PATCHES FOUND", SCREEN_WIDTH/2, SCREEN_HEIGHT/2);
    return;
  }
  
  // Draw files
  patchDir.rewindDirectory();
  int y = TOP_BAR_HEIGHT + 40;
  int fileCount = 0;
  int visibleFiles = 0;
  
  // Skip files according to scroll offset
  for (int i = 0; i < fileScrollOffset; i++) {
    entry = patchDir.openNextFile();
    if (!entry) break;
    
    if (!entry.isDirectory()) {
      const char* entryName = entry.name();
      int nameLen = strlen(entryName);
      if (nameLen > 4 && strcmp(entryName + nameLen - 4, ".kck") == 0) {
        fileCount++;
      }
    }
    entry.close();
  }
  
  // Draw visible files
  while (visibleFiles < 8) { // Show at most 8 files
    entry = patchDir.openNextFile();
    if (!entry) break;
    
    if (!entry.isDirectory()) {
      const char* entryName = entry.name();
      int nameLen = strlen(entryName);
      if (nameLen > 4 && strcmp(entryName + nameLen - 4, ".kck") == 0) {
        // Get just the filename without extension
        char filename[20];
        strncpy(filename, entryName, nameLen - 4);
        filename[nameLen - 4] = '\0';
        
        // Highlight selected file
        if (fileCount == selectedFileIndex) {
          tft.fillRect(10, y - 5, SCREEN_WIDTH - 20, 25, 0x5ACB);
          tft.setTextColor(0x0000);
        } else {
          tft.setTextColor(TEXT_COLOR);
        }
        
        tft.setTextDatum(ML_DATUM);
        tft.drawString(filename, 20, y + 8);
        
        y += 30;
        fileCount++;
        visibleFiles++;
      }
    }
    entry.close();
  }
  
  patchDir.close();
  
  // Draw up/down arrows if needed
  if (fileScrollOffset > 0) {
    // Draw up arrow
    tft.fillTriangle(SCREEN_WIDTH - 20, TOP_BAR_HEIGHT + 50, 
                    SCREEN_WIDTH - 10, TOP_BAR_HEIGHT + 35,
                    SCREEN_WIDTH - 30, TOP_BAR_HEIGHT + 35,
                    0xFFFF);
  }
  
  if (patchCount > fileScrollOffset + visibleFiles) {
    // Draw down arrow
    tft.fillTriangle(SCREEN_WIDTH - 20, SCREEN_HEIGHT - 50, 
                    SCREEN_WIDTH - 10, SCREEN_HEIGHT - 35,
                    SCREEN_WIDTH - 30, SCREEN_HEIGHT - 35,
                    0xFFFF);
  }
  
  // Draw load button if we have files
  if (patchCount > 0) {
    tft.fillRoundRect(SCREEN_WIDTH/2 - 40, SCREEN_HEIGHT - 40, 80, 30, 5, 0x04A0);
    tft.setTextColor(TEXT_COLOR);
    tft.setTextDatum(MC_DATUM);
    tft.drawString("LOAD", SCREEN_WIDTH/2, SCREEN_HEIGHT - 25);
  }
}

void processFileListTouch(int px, int py) {
  // Handle scrolling
  if (px > SCREEN_WIDTH - 40 && py < TOP_BAR_HEIGHT + 60 && fileScrollOffset > 0) {
    // Scroll up
    fileScrollOffset--;
    drawFileList();
    return;
  }
  
  if (px > SCREEN_WIDTH - 40 && py > SCREEN_HEIGHT - 60) {
    // Scroll down
    fileScrollOffset++;
    drawFileList();
    return;
  }
  
  // Handle file selection
  if (py > TOP_BAR_HEIGHT + 40 && py < SCREEN_HEIGHT - 50) {
    int fileIndex = fileScrollOffset + ((py - (TOP_BAR_HEIGHT + 40)) / 30);
    
    // Verify this is a valid file index
    bool validIndex = false;
    int count = 0;
    
    if (SD.exists("patches")) {
      patchDir = SD.open("patches");
      File entry;
      
      while (true) {
        entry = patchDir.openNextFile();
        if (!entry) break;
        
        if (!entry.isDirectory() && String(entry.name()).endsWith(".kck")) {
          if (count == fileIndex) {
            validIndex = true;
            break;
          }
          count++;
        }
        entry.close();
      }
      patchDir.close();
    }
    
    if (validIndex) {
      selectedFileIndex = fileIndex;
      drawFileList();
    }
  }
  
  // Handle load button
  if (py > SCREEN_HEIGHT - 40 && py < SCREEN_HEIGHT - 10 &&
      px > SCREEN_WIDTH/2 - 40 && px < SCREEN_WIDTH/2 + 40) {
    loadSelectedPatch();
    currentState = ENVELOPE_EDIT;
    drawUI();
  }
}

void saveCurrentPatch() {
  Serial.println("Saving patch...");
  
  // Create patches directory if needed
  if (!SD.exists("patches")) {
    SD.mkdir("patches");
  }
  
  // Construct filename - no leading slash
  char filename[50] = "patches/";
  strcat(filename, currentPatchName);
  strcat(filename, ".kck");
  
  Serial.print("Saving to filename: ");
  Serial.println(filename);
  
  // Remove existing file if it exists
  if (SD.exists(filename)) {
    Serial.println("Removing existing file");
    SD.remove(filename);
  }
  
  // Open file for writing
  File patchFile = SD.open(filename, FILE_WRITE);
  if (patchFile) {
    Serial.println("File opened for writing");
    
    // Write header
    patchFile.println("KICK_PATCH");
    patchFile.println("VERSION:1.0");
    
    // Save each envelope
    for (int e = 0; e < 4; e++) {
      patchFile.print("ENVELOPE:");
      patchFile.println(e);
      patchFile.print("DURATION:");
      patchFile.println(envelopes[e].totalDuration);
      
      // Save points
      patchFile.print("POINTS:");
      patchFile.println(envelopes[e].points.size());
      for (size_t i = 0; i < envelopes[e].points.size(); i++) {
        patchFile.print(envelopes[e].points[i].x);
        patchFile.print(",");
        patchFile.println(envelopes[e].points[i].y);
      }
      
      // Save curves
      patchFile.print("CURVES:");
      patchFile.println(envelopes[e].curves.size());
      for (size_t i = 0; i < envelopes[e].curves.size(); i++) {
        patchFile.print(envelopes[e].curves[i].index);
        patchFile.print(",");
        patchFile.println(envelopes[e].curves[i].offset);
      }
    }
    
    patchFile.close();
    Serial.println("File closed after writing");
    
    // Verify file exists
    if (SD.exists(filename)) {
      Serial.println("Verified file exists after saving");
    } else {
      Serial.println("ERROR: File does not exist after saving!");
    }
    
    // Indicate success
    tft.fillRect(SCREEN_WIDTH/4, SCREEN_HEIGHT/3, SCREEN_WIDTH/2, 40, 0x04A0);
    tft.setTextColor(TEXT_COLOR);
    tft.setTextDatum(MC_DATUM);
    tft.setTextSize(1);
    tft.drawString("PATCH SAVED", SCREEN_WIDTH/2, SCREEN_HEIGHT/3 + 20);
    delay(1000);
  } else {
    Serial.println("Failed to open file for writing!");
    // Show error message
    tft.fillRect(SCREEN_WIDTH/4, SCREEN_HEIGHT/3, SCREEN_WIDTH/2, 40, TFT_RED);
    tft.setTextColor(TEXT_COLOR);
    tft.setTextDatum(MC_DATUM);
    tft.setTextSize(1);
    tft.drawString("SAVE ERROR", SCREEN_WIDTH/2, SCREEN_HEIGHT/3 + 20);
    delay(1000);
  }
}

void loadSelectedPatch() {
  if (selectedFileIndex < 0) return;
  
  // Find the selected file by index
  if (SD.exists("patches")) {
    patchDir = SD.open("patches");
    File entry;
    int fileCount = 0;
    char filename[50] = "";
    
    while (true) {
      entry = patchDir.openNextFile();
      if (!entry) break;
      
      if (!entry.isDirectory()) {
        const char* entryName = entry.name();
        int nameLen = strlen(entryName);
        if (nameLen > 4 && strcmp(entryName + nameLen - 4, ".kck") == 0) {
          if (fileCount == selectedFileIndex) {
            strcpy(filename, entryName);
            entry.close();
            break;
          }
          fileCount++;
        }
      }
      entry.close();
    }
    patchDir.close();
    
    if (strlen(filename) > 0) {
      // Open the selected file
      char fullPath[100] = "patches/";
      strcat(fullPath, filename);
      File patchFile = SD.open(fullPath);
      
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
                envelopes[envIndex].totalDuration = atoi(buffer + 9);
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
          
          // Indicate success
          tft.fillRect(SCREEN_WIDTH/4, SCREEN_HEIGHT/3, SCREEN_WIDTH/2, 40, 0x04A0);
          tft.setTextColor(TEXT_COLOR);
          tft.setTextDatum(MC_DATUM);
          tft.setTextSize(1);
          tft.drawString("PATCH LOADED", SCREEN_WIDTH/2, SCREEN_HEIGHT/3 + 20);
          delay(1000);
        }
        
        patchFile.close();
      }
    }
  }
}
void loadDefaultPatch() {
  // Path to default patch file
  const char* defaultPatchPath = "patches/DEFAULT.kck";
  
  // Check if the default patch exists
  if (SD.exists(defaultPatchPath)) {
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
              envelopes[envIndex].totalDuration = atoi(buffer + 9);
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
        
        Serial.println("Default patch loaded successfully");
      }
      
      patchFile.close();
    }
  } else {
    Serial.println("Default patch not found");
  }
}
// Helper function to read a line from a file into a buffer
bool readLine(File &file, char* buffer, int maxLen) {
  int i = 0;
  while (file.available() && i < maxLen - 1) {
    char c = file.read();
    if (c == '\n') {
      break;
    }
    if (c == '\r') {
      continue; // Skip carriage returns
    }
    buffer[i++] = c;
  }
  buffer[i] = 0; // Null terminate
  return i > 0;  // Return true if we read anything
}

void printDirectory(File dir, int numTabs) {
  while (true) {
    File entry = dir.openNextFile();
    if (!entry) {
      break; // No more files
    }
    
    for (int i = 0; i < numTabs; i++) {
      Serial.print('\t');
    }
    
    Serial.print(entry.name());
    
    if (entry.isDirectory()) {
      Serial.println("/");
      printDirectory(entry, numTabs + 1);
    } else {
      // Files have sizes, directories do not
      Serial.print("\t\t");
      Serial.println(entry.size(), DEC);
    }
    entry.close();
  }
}

void processTouch() {
  if (ts.touched()) {
    TS_Point p = ts.getPoint();
    int px = map(p.x, 3900, 300, -5, 315);
    int py = map(p.y, 3900, 300, 0, 240);
    px = constrain(stableX(px), 0, SCREEN_WIDTH);
    py = constrain(stableY(py), 0, SCREEN_HEIGHT);
    
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
    if (isInTimeSlider(px, py)) {
    // Initial touch on the slider
    draggingTimeSlider = true;
    int newPos = constrain(px - TIME_SLIDER_X, 0, TIME_SLIDER_WIDTH);
    int newTime = map(newPos, 0, TIME_SLIDER_WIDTH, TIME_MIN_VALUE, TIME_MAX_VALUE);
    envelopes[currentEnvelope].totalDuration = newTime;
    drawUI(false);
    return;  // Important: return to prevent other touch handling
  }
}
    // Handle different UI states
    if (currentState == SAVE_NAME_ENTRY) {
      processKeyboardTouch(px, py);
      return;
    } else if (currentState == LOAD_FILE_SELECT) {
      processFileListTouch(px, py);
      return;
    }
    
    // Original envelope editing code
    // Time slider handling - keep this section near the top of your processTouch function
// so it happens before point/curve handling
if (currentState == ENVELOPE_EDIT) {
  if (draggingTimeSlider) {
    // If we're dragging, update the slider position
    int newPos = constrain(px - TIME_SLIDER_X, 0, TIME_SLIDER_WIDTH);
    int newTime = map(newPos, 0, TIME_SLIDER_WIDTH, TIME_MIN_VALUE, TIME_MAX_VALUE);
    envelopes[currentEnvelope].totalDuration = newTime;
    drawUI(false);  // Partial redraw
    return;  // Important: return to prevent other touch handling
  } else 
    
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
            float lineLength = sqrt(dx * dx + dy * dy);
            
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
