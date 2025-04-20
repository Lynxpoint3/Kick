#include <TFT_eSPI.h>
#include <XPT2046_Touchscreen.h>
#include <Wire.h>
#include <Adafruit_MCP4728.h>
#include <vector>

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
#define TAB_WIDTH 80
#define TAB_HEIGHT 30
#define MAX_POINTS 20
#define POINT_RADIUS 6
#define LINE_SELECT_RADIUS 15
#define HOLD_DELAY 300
#define JITTER_BUFFER_SIZE 4
#define TIME_SLIDER_WIDTH 180
#define TIME_SLIDER_HEIGHT 20
#define TIME_SLIDER_X 70
#define TIME_SLIDER_Y (SCREEN_HEIGHT - 20) // Place slider near bottom
#define DOUBLE_CLICK_TIME 300 // Time in ms to detect a double-click

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
const uint16_t ENV_COLORS[] = {0x07FF, 0x07E0, 0xFD20, 0xF800}; // Cyan, Green, Yellow, Red

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
int selectedPoint = -1;
int selectedCurve = -1;
bool holdingPoint = false;
bool holdingCurve = false;
bool draggingTimeSlider = false;
unsigned long holdStartTime = 0;
unsigned long lastClickTime = 0;
int lastClickedPoint = -1;
std::vector<int> jitterBufferX;
std::vector<int> jitterBufferY;

// Function prototypes
void drawUI();
void drawEnvelope(EnvelopeType envType);
void drawTabs();
void drawTimeSlider();
void drawGrid();
int findNearbyPoint(int x, int y);
int findNearbyLine(int x, int y);
void processTouch();
void updateOutputs();
void removeLineOverlaps();
int stableX(int rawX);
int stableY(int rawY);

void setup() {
  Serial.begin(115200);
  tft.init();
  tft.setRotation(3);
  tft.fillScreen(BG_COLOR);
  ts.begin();
  ts.setRotation(1);
  if (!dac.begin()) {
    Serial.println("Failed to find MCP4728 chip");
  }
  drawUI();
}

void loop() {
  processTouch();
  updateOutputs();
  delay(10);
}

void drawUI() {
  tft.fillScreen(BG_COLOR);
  drawGrid();
  drawEnvelope(currentEnvelope);
  drawTabs();
  drawTimeSlider();
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
  tft.setTextColor(TEXT_COLOR, BG_COLOR);
  tft.setTextDatum(TR_DATUM);
  tft.setTextSize(1);
  char durationText[20];
  if (env.totalDuration >= 1000) {
    sprintf(durationText, "LENGTH: %.2f s", env.totalDuration / 1000.0);
  } else {
    sprintf(durationText, "LENGTH: %d ms", env.totalDuration);
  }
  tft.drawString(durationText, SCREEN_WIDTH - 5, ENV_VIEW_TOP + 15);
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
      int cx = (p1.x + p2.x) / 2 + curveOffset;
      int cy = (p1.y + p2.y) / 2;
      for (int t = 0; t <= 20; t++) {
        float u = t / 20.0;
        float v = (t+1) / 20.0;
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

void drawTabs() {
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(MC_DATUM);
  tft.setTextSize(1);
  for (int i = 0; i < 4; i++) {
    int x = i * TAB_WIDTH;
    int y = TOP_BAR_HEIGHT / 2;
    uint16_t tabColor = (i == currentEnvelope) ? TAB_ACTIVE : TAB_INACTIVE;
    uint16_t lineColor = ENV_COLORS[i];
    tft.fillRoundRect(x + 2, 2, TAB_WIDTH - 4, TOP_BAR_HEIGHT - 4, 4, tabColor);
    if (i == currentEnvelope) {
      tft.drawLine(x + 2, TOP_BAR_HEIGHT - 2, x + TAB_WIDTH - 2, TOP_BAR_HEIGHT - 2, lineColor);
    }
    tft.drawString(ENV_NAMES[i], x + TAB_WIDTH/2, y);
  }
}

void drawTimeSlider() {
  Envelope &env = envelopes[currentEnvelope];
  int sliderPos = map(env.totalDuration, 100, 5000, 0, TIME_SLIDER_WIDTH);
  tft.fillRoundRect(TIME_SLIDER_X, TIME_SLIDER_Y - TIME_SLIDER_HEIGHT/2, 
                   TIME_SLIDER_WIDTH, TIME_SLIDER_HEIGHT, 
                   TIME_SLIDER_HEIGHT/2, SLIDER_BG);
  tft.fillRoundRect(TIME_SLIDER_X, TIME_SLIDER_Y - TIME_SLIDER_HEIGHT/2, 
                   sliderPos, TIME_SLIDER_HEIGHT, 
                   TIME_SLIDER_HEIGHT/2, SLIDER_FG);
  tft.setTextColor(TEXT_COLOR);
  tft.setTextDatum(MR_DATUM);
  tft.setTextSize(1);
  tft.drawString("TIME:", TIME_SLIDER_X - 5, TIME_SLIDER_Y);
}

int findNearbyPoint(int x, int y) {
  Envelope &env = envelopes[currentEnvelope];
  for (size_t i = 0; i < env.points.size(); i++) {
    int dx = env.points[i].x - x;
    int dy = env.points[i].y - y;
    if (dx*dx + dy*dy <= POINT_RADIUS*POINT_RADIUS) {
      return i;
    }
  }
  return -1;
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
  return x >= TIME_SLIDER_X && x <= TIME_SLIDER_X + TIME_SLIDER_WIDTH &&
         y >= TIME_SLIDER_Y - TIME_SLIDER_HEIGHT && y <= TIME_SLIDER_Y + TIME_SLIDER_HEIGHT;
}

int getEnvelopeTabAt(int x, int y) {
  if (y <= TOP_BAR_HEIGHT) {
    for (int i = 0; i < 4; i++) {
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

void processTouch() {
  if (ts.touched()) {
    TS_Point p = ts.getPoint();
    int px = map(p.x, 3900, 300, -5, 315);
    int py = map(p.y, 3900, 300, 0, 240);
    px = constrain(stableX(px), 0, SCREEN_WIDTH);
    py = constrain(stableY(py), 0, SCREEN_HEIGHT);
    int tabIndex = getEnvelopeTabAt(px, py);
    if (tabIndex >= 0) {
      if (tabIndex != currentEnvelope) {
        currentEnvelope = (EnvelopeType)tabIndex;
        selectedPoint = -1;
        selectedCurve = -1;
        holdingPoint = false;
        holdingCurve = false;
        lastClickedPoint = -1;
        drawUI();
      }
      return;
    }
    if (isInTimeSlider(px, py)) {
      int sliderValue = map(px - TIME_SLIDER_X, 0, TIME_SLIDER_WIDTH, 100, 5000);
      envelopes[currentEnvelope].totalDuration = constrain(sliderValue, 100, 5000);
      drawUI();
      return;
    }
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
        } else if (selectedPoint == static_cast<int>(env.points.size()) - 1) { // Fix signedness warning
          env.points[selectedPoint].x = SCREEN_WIDTH - 1;
        }
        removeLineOverlaps(); // Handles drag-over deletion
        drawUI();
      } else if (holdingCurve) {
        for (auto &c : env.curves) {
          if (c.index == selectedCurve) {
            c.offset = px - (env.points[c.index].x + env.points[c.index + 1].x) / 2;
            drawUI();
            break;
          }
        }
      } else {
        int pointIndex = findNearbyPoint(px, py);
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
        // This code only executes if we didn't find a nearby point or line
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
