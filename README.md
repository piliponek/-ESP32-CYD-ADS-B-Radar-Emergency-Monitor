✈️ ESP32 CYD ADS-B Radar & Emergency Monitor
W pełni autonomiczny, stacjonarny radar ADS-B i monitor sytuacji w powietrzu, zaprojektowany specjalnie dla entuzjastów lotnictwa oraz radioamatorów. Projekt uruchomiony jest na popularnej, niskobudżetowej płytce Sunton ESP32-035 (Cheap Yellow Display - CYD 3.5") i wykorzystuje wydajny ekosystem ESPHome / ESP-IDF.
System pobiera dane w czasie rzeczywistym z lokalnego serwera ADS-B (np. Dump1090 / Readsb) w sieci domowej, wyświetlając dynamiczną tabelę przelotów oraz interaktywną mapę radaru o zasięgu do 150 km.
---
🛠️ Specyfikacja Sprzętowa i Systemowa
Mikrokontroler: Sunton ESP32-035 (ESP32).
Ekran: 3.5" TFT LCD (sterownik ST7796) o rozdzielczości $480 \times 320$ px.
Panel Dotykowy: XPT2046 z obsługą przerwań sprzętowych (`GPIO36`) i pełną kalibracją osi.
Framework: Natywny ESP-IDF działający pod kontrolą ESPHome.
Komunikacja: Pobieranie danych JSON przez HTTP w interwale 10-sekundowym.
---
💡 Kluczowe Funkcje Oprogramowania
Dynamiczna Tabela Samolotów (Multi-page):
Wyświetlanie listy wykrytych maszyn z uwzględnieniem znaków rozpoznawczych (Ident/Flight), kodów Squawk, wysokości (w metrach), prędkości (w km/h) oraz dystansu od domowej lokalizacji (w km).
Automatyczna paginacja dostosowująca się do liczby wykrytych celów.
Interaktywna Strona Mapy Radaru (150 km):
Wizualizacja radarowa w układzie biegunowym z okręgami odniesienia na dystansie 50 km, 100 km oraz 150 km.
Naniesione pozycje samolotów względem domowej lokalizacji QTH ($52.0000^\circ\text{N}$, $15.0000^\circ\text{E}$).
Zaawansowany System Powiadomień Alarmowych (Squawk 7700 / 7600):
Błyskawiczne wykrycie kodów awaryjnych:
Squawk 7700: Ogólne zagrożenie / stan nagły (Emergency).
Squawk 7600: Awaria łączności radiowej (Radio Failure).
Pełnoekranowy, czerwony alert wizualny z informacją o locie i kodzie squawk, z możliwością wyciszenia dotknięciem ekranu.
Interaktywne Okno Szczegółów Lotu:
Dotknięcie ikony samolotu bezpośrednio na ekranie mapy powoduje otwarcie okna dialogowego z pełnymi parametrami wybranej maszyny.
Sterowanie Dotykowe i Pauza:
Kliknięcie w lewy górny róg ekranu (`x < 100`, `y < 40`) wstrzymuje automatyczną rotację stron (`[ PAUZA ]`), ułatwiając dokładną analizę danych.
---Projekt udostępniony w celach edukacyjnych oraz dla społeczności krótkofalarskiej. 73 de SP3PM! 
📂 Plik Konfiguracyjny ESPHome (`cyd-adsb-radar.yaml`)
Kompletny, zoptymalizowany kod konfiguracyjny dla oprogramowania ESPHome:
```yaml
esphome:
  name: cyd-adsb-radar
  friendly_name: "CYD ADS-B Radar"
  on_boot:
    priority: -10.0
    then:
      - lambda: |-
          id(plane_list).clear();
          id(current_page) = 0;
          id(emergency_active) = false;
          id(selected_plane_active) = false;

esp32:
  board: esp32dev
  framework:
    type: esp-idf

wifi:
  ssid: "Twoje_WiFi"
  password: "Twoje_Haslo"
  ap:
    ssid: "CYD-ADSB-Fallback"
    password: "fallbackpassword"

captive_portal:

logger:
  level: INFO

api:

ota:
  - platform: esphome
    password: Twoje_Haslo

json:

http_request:
  timeout: 10s
  follow_redirects: true

globals:
  - id: total_aircraft
    type: int
    restore_value: no
    initial_value: '0'
  - id: aircraft_with_pos
    type: int
    restore_value: no
    initial_value: '0'
  - id: plane_list
    type: std::vector<std::string>
    restore_value: no
  - id: current_page
    type: int
    restore_value: no
    initial_value: '0'
  - id: emergency_active
    type: bool
    restore_value: no
    initial_value: 'false'
  - id: emergency_details
    type: std::string
    restore_value: no
    initial_value: '""'
  - id: selected_plane_active
    type: bool
    restore_value: no
    initial_value: 'false'
  - id: sel_flight
    type: std::string
    restore_value: no
    initial_value: '""'
  - id: sel_squawk
    type: std::string
    restore_value: no
    initial_value: '""'
  - id: sel_alt
    type: std::string
    restore_value: no
    initial_value: '""'
  - id: sel_spd
    type: std::string
    restore_value: no
    initial_value: '""'
  - id: sel_dist
    type: std::string
    restore_value: no
    initial_value: '""'

spi:
  - id: tft_spi
    clk_pin: GPIO14
    mosi_pin: GPIO13
    miso_pin: GPIO12

touchscreen:
  - platform: xpt2046
    id: cyd_touch
    spi_id: tft_spi
    cs_pin: GPIO33
    interrupt_pin: GPIO36
    update_interval: 50ms
    threshold: 400
    calibration:
      x_min: 280
      x_max: 3860
      y_min: 340
      y_max: 3860
    transform:
      swap_xy: true
      mirror_x: true
      mirror_y: false
    on_touch:
      then:
        - lambda: |-
            int touch_x = touch.x;
            int touch_y = touch.y;

            if (id(emergency_active)) {
              id(emergency_active) = false;
              return;
            }

            if (id(selected_plane_active)) {
              id(selected_plane_active) = false;
              return;
            }

            if (touch_x < 100 && touch_y < 40) {
              if (id(automatyczna_rotacja).state) {
                id(automatyczna_rotacja).turn_off();
              } else {
                id(automatyczna_rotacja).turn_on();
              }
              return;
            }

            int total_planes = id(plane_list).size();
            int max_rows = 7;
            int table_pages = (total_planes + max_rows - 1) / max_rows;
            if (table_pages < 1) table_pages = 1;

            if (id(current_page) == table_pages) {
              int cx = 240;
              int cy = 150;
              float max_range_km = 150.0;
              float max_radius_px = 135.0;
              float home_lat = 52.8405;
              float home_lon = 15.83035;

              for (const auto& plane_str : id(plane_list)) {
                size_t p1 = plane_str.find('|');
                size_t p2 = plane_str.find('|', p1 + 1);
                size_t p3 = plane_str.find('|', p2 + 1);
                size_t p4 = plane_str.find('|', p3 + 1);
                size_t p5 = plane_str.find('|', p4 + 1);
                size_t p6 = plane_str.find('|', p5 + 1);

                if (p6 == std::string::npos) continue;

                std::string flight = plane_str.substr(0, p1);
                std::string squawk = plane_str.substr(p1 + 1, p2 - p1 - 1);
                std::string alt = plane_str.substr(p2 + 1, p3 - p2 - 1);
                std::string spd = plane_str.substr(p3 + 1, p4 - p3 - 1);
                std::string dist = plane_str.substr(p4 + 1, p5 - p4 - 1);
                float lat = std::stof(plane_str.substr(p5 + 1, p6 - p5 - 1));
                float lon = std::stof(plane_str.substr(p6 + 1));

                float dx_km = (lon - home_lon) * 111.32 * cos(home_lat * M_PI / 180.0);
                float dy_km = (lat - home_lat) * 111.32;

                float px_offset = dx_km * (max_radius_px / max_range_km);
                float py_offset = -dy_km * (max_radius_px / max_range_km);

                int screen_x = cx + (int)px_offset;
                int screen_y = cy + (int)py_offset;

                if (abs(touch_x - screen_x) <= 45 && abs(touch_y - screen_y) <= 45) {
                  id(sel_flight) = flight;
                  id(sel_squawk) = squawk;
                  id(sel_alt) = alt;
                  id(sel_spd) = spd;
                  id(sel_dist) = dist;
                  id(selected_plane_active) = true;
                  return;
                }
              }
            }

switch:
  - platform: gpio
    pin: GPIO27
    id: backlight_switch
    name: "Podswietlenie"
    restore_mode: ALWAYS_ON

  - platform: template
    name: "Automatyczna Rotacja Stron"
    id: automatyczna_rotacja
    optimistic: true
    restore_mode: ALWAYS_ON

interval:
  - interval: 5s
    then:
      - if:
          condition:
            switch.is_on: automatyczna_rotacja
          then:
            - lambda: |-
                if (!id(emergency_active) && !id(selected_plane_active)) {
                  int total_planes = id(plane_list).size();
                  int max_rows = 7;
                  int table_pages = (total_planes + max_rows - 1) / max_rows;
                  if (table_pages < 1) table_pages = 1;
                  
                  int total_display_pages = table_pages + 1;
                  id(current_page) = (id(current_page) + 1) % total_display_pages;
                }

  - interval: 10s
    then:
      - http_request.get:
          url: "http://192.168.1.252:8080/data/aircraft.json"
          capture_response: true
          max_response_buffer_size: 32768
          on_response:
            then:
              - if:
                  condition:
                    lambda: |-
                      return response->status_code == 200;
                  then:
                    - lambda: |-
                        if (body.empty() || body.find("{") == std::string::npos) return;

                        struct LocalPlane {
                          std::string flight;
                          std::string squawk;
                          std::string alt;
                          std::string spd;
                          std::string dist;
                          float dist_val;
                          float lat;
                          float lon;
                        };

                        json::parse_json(body, [](JsonObject root) -> bool {
                          if (!root["aircraft"].is<JsonArray>()) return false;
                          JsonArray aircraft = root["aircraft"];
                          int total = aircraft.size();
                          int with_pos = 0;
                          
                          std::vector<LocalPlane> temp_planes;
                          float home_lat = 52.0000;
                          float home_lon = 15.0000;

                          bool found_emergency = false;
                          std::string emergency_info = "";

                          for (JsonVariant v : aircraft) {
                            JsonObject ac = v.as<JsonObject>();
                            
                            std::string squawk = ac["squawk"].is<const char*>() ? ac["squawk"].as<std::string>() : "----";
                            std::string flight = ac["flight"].is<const char*>() ? ac["flight"].as<std::string>() : "N/A";
                            flight.erase(flight.find_last_not_of(" \n\r\t")+1);
                            if(flight.empty()) flight = "N/A";

                            if (squawk == "7700" || squawk == "7600") {
                              found_emergency = true;
                              std::string alert_type = (squawk == "7700") ? "ALERT 7700 (ZAGROZENIE)" : "ALERT 7600 (RADIO FAIL)";
                              emergency_info = alert_type + "\nLot: " + flight + "\nSquawk: " + squawk;
                            }

                            if (ac["lat"].is<float>() && ac["lon"].is<float>()) {
                              with_pos++;
                              float lat = ac["lat"].as<float>();
                              float lon = ac["lon"].as<float>();

                              float dLat = (lat - home_lat) * M_PI / 180.0;
                              float dLon = (lon - home_lon) * M_PI / 180.0;
                              float lat1 = home_lat * M_PI / 180.0;
                              float lat2 = lat * M_PI / 180.0;
                              float a = sin(dLat / 2) * sin(dLat / 2) + sin(dLon / 2) * sin(dLon / 2) * cos(lat1) * cos(lat2);
                              float c = 2 * atan2(sqrt(a), sqrt(1 - a));
                              float distance_km = 6371.0 * c;

                              std::string alt = "---";
                              if (ac["alt_baro"].is<int>()) {
                                int alt_m = (int)(ac["alt_baro"].as<int>() * 0.3048);
                                alt = std::to_string(alt_m);
                              }

                              std::string spd = "---";
                              if (ac["gs"].is<float>()) {
                                int spd_kmh = (int)(ac["gs"].as<float>() * 1.852);
                                spd = std::to_string(spd_kmh);
                              }

                              char dist_buf[32];
                              snprintf(dist_buf, sizeof(dist_buf), "%.1f", distance_km);

                              temp_planes.push_back({flight, squawk, alt, spd, std::string(dist_buf), distance_km, lat, lon});
                            }
                          }

                          std::sort(temp_planes.begin(), temp_planes.end(), [](const LocalPlane& a, const LocalPlane& b) {
                            return a.dist_val < b.dist_val;
                          });

                          std::vector<std::string> final_list;
                          for (const auto& p : temp_planes) {
                            char buf[256];
                            snprintf(buf, sizeof(buf), "%s|%s|%s|%s|%s|%.6f|%.6f", 
                              p.flight.c_str(), p.squawk.c_str(), p.alt.c_str(), 
                              p.spd.c_str(), p.dist.c_str(), p.lat, p.lon);
                            final_list.push_back(std::string(buf));
                          }

                          id(total_aircraft) = total;
                          id(aircraft_with_pos) = with_pos;
                          id(plane_list) = final_list;
                          id(emergency_active) = found_emergency;
                          if (found_emergency) {
                            id(emergency_details) = emergency_info;
                          }

                          return true;
                        });

display:
  - platform: mipi_spi
    id: cyd_display
    model: ST7796
    cs_pin: GPIO15
    dc_pin: GPIO02
    spi_id: tft_spi
    rotation: 90
    update_interval: 1s
    data_rate: 40MHz
    lambda: |-
      Color c_black(0, 0, 0);
      Color c_yellow(255, 255, 0);
      Color c_red(255, 0, 0);
      Color c_gray(128, 128, 128);
      Color c_white(255, 255, 255);
      Color c_cyan(0, 255, 255);
      Color c_green(0, 255, 0);
      Color c_light_gray(200, 200, 200);

      it.fill(c_black);

      if (id(emergency_active)) {
        it.rectangle(15, 15, 450, 290, c_red);
        it.rectangle(16, 16, 448, 288, c_red);
        it.print(240, 30, id(font_bold), c_red, TextAlign::TOP_CENTER, "!!! WYKRYTO SYGNAL ALARMOWY !!!");
        it.filled_rectangle(30, 70, 420, 150, Color(50, 0, 0));
        it.rectangle(30, 70, 420, 150, c_red);
        it.print(240, 95, id(font_bold), c_yellow, TextAlign::TOP_CENTER, id(emergency_details).c_str());
        it.print(240, 230, id(font_regular), c_white, TextAlign::TOP_CENTER, "Dotknij ekranu aby wylaczyc alarm");
        it.print(240, 255, id(font_small), c_light_gray, TextAlign::TOP_CENTER, "Sprawdz dane radaru / flightradara");
        return;
      }

      int total_planes = id(plane_list).size();
      int max_rows = 7;
      int table_pages = (total_planes + max_rows - 1) / max_rows;
      if (table_pages < 1) table_pages = 1;
      int total_display_pages = table_pages + 1;

      if (id(current_page) >= total_display_pages) {
        id(current_page) = 0;
      }

      if (id(current_page) == table_pages) {
        int cx = 240;
        int cy = 150; 
        float max_range_km = 150.0;
        float max_radius_px = 135.0; 

        it.circle(cx, cy, 45, c_gray);
        it.circle(cx, cy, 90, c_gray);
        it.circle(cx, cy, (int)max_radius_px, c_cyan);

        it.line(cx, cy - (int)max_radius_px - 4, cx, cy + (int)max_radius_px + 4, c_gray);
        it.line(cx - (int)max_radius_px - 4, cy, cx + (int)max_radius_px + 4, cy, c_gray);

        it.print(cx + 4, cy - 45, id(font_small), c_gray, TextAlign::TOP_LEFT, "50km");
        it.print(cx + 4, cy - 90, id(font_small), c_gray, TextAlign::TOP_LEFT, "100km");
        it.print(cx + 4, cy - (int)max_radius_px, id(font_small), c_cyan, TextAlign::TOP_LEFT, "150km");

        it.filled_circle(cx, cy, 3, c_green);
        it.print(cx, cy + 5, id(font_small), c_green, TextAlign::TOP_CENTER, "DOM");

        float home_lat = 52.8405;
        float home_lon = 15.83035;

        for (const auto& plane_str : id(plane_list)) {
          size_t p1 = plane_str.find('|');
          size_t p2 = plane_str.find('|', p1 + 1);
          size_t p3 = plane_str.find('|', p2 + 1);
          size_t p4 = plane_str.find('|', p3 + 1);
          size_t p5 = plane_str.find('|', p4 + 1);
          size_t p6 = plane_str.find('|', p5 + 1);

          if (p6 == std::string::npos) continue;

          std::string flight = plane_str.substr(0, p1);
          std::string squawk = plane_str.substr(p1 + 1, p2 - p1 - 1);
          float lat = std::stof(plane_str.substr(p5 + 1, p6 - p5 - 1));
          float lon = std::stof(plane_str.substr(p6 + 1));

          float dx_km = (lon - home_lon) * 111.32 * cos(home_lat * M_PI / 180.0);
          float dy_km = (lat - home_lat) * 111.32; 

          float px_offset = dx_km * (max_radius_px / max_range_km);
          float py_offset = -dy_km * (max_radius_px / max_range_km);

          int screen_x = cx + (int)px_offset;
          int screen_y = cy + (int)py_offset;

          if (abs(px_offset) <= max_radius_px && abs(py_offset) <= max_radius_px) {
            Color dot_color = (squawk == "7700" || squawk == "7600") ? c_red : c_yellow;
            it.filled_circle(screen_x, screen_y, 2, dot_color);
            it.print(screen_x + 4, screen_y - 4, id(font_small), c_white, TextAlign::TOP_LEFT, flight.c_str());
          }
        }

        if (id(selected_plane_active)) {
          it.filled_rectangle(60, 45, 360, 210, Color(20, 30, 50));
          it.rectangle(60, 45, 360, 210, c_cyan);
          it.rectangle(61, 46, 358, 208, c_cyan);
          
          it.print(240, 55, id(font_bold), c_yellow, TextAlign::TOP_CENTER, "SZCZEGOLY WYBRANEGO LOTU");
          
          it.printf(80, 90, id(font_regular), c_white, TextAlign::TOP_LEFT, "Lot: %s", id(sel_flight).c_str());
          it.printf(80, 115, id(font_regular), c_white, TextAlign::TOP_LEFT, "Squawk: %s", id(sel_squawk).c_str());
          it.printf(80, 140, id(font_regular), c_white, TextAlign::TOP_LEFT, "Wysokosc: %s m", id(sel_alt).c_str());
          it.printf(80, 165, id(font_regular), c_white, TextAlign::TOP_LEFT, "Predkosc: %s km/h", id(sel_spd).c_str());
          it.printf(80, 190, id(font_regular), c_white, TextAlign::TOP_LEFT, "Dystans: %s km", id(sel_dist).c_str());
          
          it.print(240, 225, id(font_small), c_light_gray, TextAlign::TOP_CENTER, "Dotknij ekranu aby zamknac");
        }

        if (!id(automatyczna_rotacja).state && !id(selected_plane_active)) {
          it.print(10, 5, id(font_small), c_red, TextAlign::TOP_LEFT, "[ PAUZA ]");
        }

        it.printf(10, 298, id(font_small), c_light_gray, TextAlign::TOP_LEFT, "Wszystkie: %d | Z poz.: %d", id(total_aircraft), id(aircraft_with_pos));
        it.printf(470, 298, id(font_small), c_cyan, TextAlign::TOP_RIGHT, "STRONA %d/%d (MAPA MAX 150km)", id(current_page) + 1, total_display_pages);
        return;
      }

      it.print(240, 5, id(font_bold), c_yellow, TextAlign::TOP_CENTER, "RADAR ADS-B (Wszystkie cele)");
      
      if (!id(automatyczna_rotacja).state) {
        it.print(10, 5, id(font_small), c_red, TextAlign::TOP_LEFT, "[ PAUZA ]");
      }

      it.line(10, 25, 470, 25, c_gray);

      it.print(10, 32, id(font_small), c_cyan, TextAlign::TOP_LEFT, "Ident");
      it.print(105, 32, id(font_small), c_cyan, TextAlign::TOP_LEFT, "Squawk");
      it.print(200, 32, id(font_small), c_cyan, TextAlign::TOP_LEFT, "Alt(m)");
      it.print(290, 32, id(font_small), c_cyan, TextAlign::TOP_LEFT, "Spd(km/h)");
      it.print(395, 32, id(font_small), c_cyan, TextAlign::TOP_LEFT, "Dist(km)");
      it.line(10, 50, 470, 50, c_gray);

      int start_index = id(current_page) * max_rows;
      int y_offset = 58;
      int displayed = 0;

      for (int i = start_index; i < total_planes && displayed < max_rows; i++) {
        std::string plane_str = id(plane_list)[i];

        size_t pos1 = plane_str.find('|');
        size_t pos2 = plane_str.find('|', pos1 + 1);
        size_t pos3 = plane_str.find('|', pos2 + 1);
        size_t pos4 = plane_str.find('|', pos3 + 1);

        std::string flight = plane_str.substr(0, pos1);
        std::string squawk = plane_str.substr(pos1 + 1, pos2 - pos1 - 1);
        std::string alt = plane_str.substr(pos2 + 1, pos3 - pos2 - 1);
        std::string spd = plane_str.substr(pos3 + 1, pos4 - pos3 - 1);
        std::string dist = plane_str.substr(pos4 + 1, plane_str.find('|', pos4 + 1) - pos4 - 1);

        Color row_color = (squawk == "7700" || squawk == "7600") ? c_red : c_yellow;

        it.print(10, y_offset, id(font_regular), row_color, TextAlign::TOP_LEFT, flight.c_str());
        it.print(105, y_offset, id(font_regular), (squawk == "7700" || squawk == "7600") ? c_red : c_green, TextAlign::TOP_LEFT, squawk.c_str());
        it.print(200, y_offset, id(font_regular), c_white, TextAlign::TOP_LEFT, alt.c_str());
        it.print(290, y_offset, id(font_regular), c_white, TextAlign::TOP_LEFT, spd.c_str());
        it.print(395, y_offset, id(font_regular), c_cyan, TextAlign::TOP_LEFT, dist.c_str());

        y_offset += 28;
        displayed++;
      }

      if (total_planes == 0) {
        it.print(240, 110, id(font_regular), c_gray, TextAlign::TOP_CENTER, "Brak samolotow w zasiegu");
      }

      it.line(10, 260, 470, 260, c_gray);

      it.printf(10, 270, id(font_small), c_light_gray, TextAlign::TOP_LEFT, "Wszystkie: %d | Z poz.: %d", id(total_aircraft), id(aircraft_with_pos));
      it.printf(470, 270, id(font_small), c_light_gray, TextAlign::TOP_RIGHT, "Strona %d/%d (Mapa)", id(current_page) + 1, total_display_pages);

font:
  - file: "gfonts://Roboto"
    id: font_regular
    size: 16
  - file: "gfonts://Roboto"
    id: font_bold
    size: 18
  - file: "gfonts://Roboto"
    id: font_small
    size: 12
```
---
🚀 Licencja
Projekt udostępniony w celach hobbystycznych i edukacyjnych.
