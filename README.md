# Hitachi Yutaki M-series heatpump H-link data bus opening with esp8266
Digging out Yutaki M-series (RHUE-XX) H-link bus for reading info to Home assistant. English vesion available. This is AI-aided test project. Surprisingly very little part needs corrections.

# Yutaki M-sarjan H-Link väylän purku ESP8266:lla ja integraatio Home Assistantiin

**Tekijä: Jussi**  
**GitHub-tunnus: [jkirjo](https://github.com/jkirjo)**  
**Lämpöpumppumalli: Hitachi Yutaki M RHUE5AVN**  

Tämä projekti kuvaa, kuinka Hitachin Yutaki M -sarjan lämpöpumpun H-Link -väylästä saadaan purettua reaaliaikaisia tietoja ESP8266:n avulla ja siirrettyä ne Home Assistantiin. Ratkaisu on testattu käytännössä, ja se tarjoaa valvontakäyttöön näkymän mm. lämpötiloihin, paineisiin ja kompressorin tilaan — ilman virallista dokumentaatiota.
!! Oleellinen lisäys käytössä havaitusta !!
Jos laite jostain syystä joutuu vikatilaan, niin tuon väylän datan rakenne muuttuu oleellisesti ja saadut arvot ovat ihan jotain muuta kuin tarkoitetut. Vikatila ei aina poistu edes sammuttamalla ja virtabootilla siitä väylän datasta heti ensimmäisellä kerralla.
Siihen on mulla paikkaus jo olemassa esp-palikan softaan, mutta siitäkin huolimatta ainakin tilatavu tulee väärin vikatilan jälkeen hetken. Kun tilatavun paikka oikeassa viestissä pysyy, mutta sisältö vaihtuu. Se muuttuu havaitusti ainakin niin, että matalapainesuojauksen vikatila näkyykin sulatuksena. Mitä se ei ole.

## 🧠 Tausta ja tarkoitus

Hitachi ei dokumentoi H-Link -väylän rakennetta eikä tarjoa virallista API:a. Tämän projektin tarkoitus on tarjota toimiva ja kenttätestattu tapa:
- Kuunnella väylää ilman vaikutusta järjestelmän toimintaan
- Purkaa 9600N1-muotoista sarjadataa paketiksi
- Tulkita kriittisiä tavuarvoja lämpöpumpun tilasta
- Viedä nämä tiedot Home Assistantin kojelautaan helposti luettavina antureina

Ohjaus ei ole mahdollista, mutta valvonta toimii erinomaisesti.

## 📡 Väylän rakenne ja purkupiste

H-Link käyttää 9600 bps sarjaliikennettä, joka on moduloitu 9,6kHz kanttiaallolla. Helpoiten signaali saadaan puhtaana väyläpiirin **RX-piirin jälkeen olevan LED:n toisesta karvasta** – sieltä löytyy valmis, puhdistettu 9600N1-data. Tämä led on se alin keltainen, joka vilahtelee välillä.

Käytettynä oli **ESP8266** -sarjan mikrokontrolleri, joka liitettiin vain vastaanottavaksi lukijaksi (`TX` johdin jätetty kytkemättä tai irti testitilassa). Väylä vaatii toimiakseen laitteeseen asennetun **system controller -kortin**, joka varmistaa että emolevy aktivoi väylän. Väylälle lähetys voi toimia ilman H-link lisäkorttiakin, se näkyy jos emolevyn alin keltainen led vilahtelee toisinaan.

## 🧰 Kaapelointi

- ESP8266 RX → LED signaalin suodatettu karva. Tässä varauksella turvallinen tapa kytkeä outoon laitteeseen, jossa siis on ylösvetovastus 3V3:sta ja diodi luettavaan värkkiin päin. Ei tahdo riittää nollataso
  diodin kanssa. Pitäisi olla ehkä schottky-diodi. Olen käyttänyt ilman diodia, ja on kestänyt nyt jo pitkään.
- TX → Ei kytketä väylään
- GND yhdistetty samaan maahan väyläpiirin kanssa

## 📦 Datapaketin formaatti

Raakapaketti standby, mutta ei käynnissä:
3F:01:09:03:1F:E0:C0:00:03:1C:FC:F3:06:F3:01:30:01:00:00:FF:19:1E:00:00:0F:00:06:68:00:00:40:01:00:00:00:00:00:03:03:02:80:80:00:00:00:00:00:00:00:80:00:2D:16:04:10:03:00:00:03:00:45

Raakapaketti käynnissä:
3F:01:09:03:1F:E0:C0:00:00:03:FF:F3:06:F3:01:30:01:00:00:FF:19:1E:00:00:0F:00:06:68:00:00:40:01:00:80:26:01:00:15:1A:08:80:80:00:00:00:90:00:00:4F:80:00:27:78:05:41:05:01:AD:27:00:82

Tuossa on vielä selvittämättömiä tavuja.

 **61 tavua pitkä paketti, joka alkaa tavuilla 3F:01:09** on olennainen.

Tärkeimmät tavut tässä paketissa:

| Tavu | Selitys                      | Muoto            |
|------|------------------------------|------------------|
| 22   | Lämmityksen pyynti           | Celsius          |
| 34   | Tilatieto (tila/vika)        | Hex -> teksti    |
| 38   | Kiertovesi sisään            | Celsius          |
| 39   | Kiertovesi ulos              | Celsius          |
| 40   | Ulkolämpötila                | Celsius          |
| 49   | Kompressorin taajuus         | Hz               |
| 52   | Imupaine                     | Bar (arvo /10)   |
| 53   | Korkeapaine (laudutin)       | Bar (arvo /5)    |
| 55   | Kuumakaasun lämpötila        | Celsius          |
| 56   | Imulinjan lämpö              | Celsius          |
| 57+58| EEV venttiilin avautuma      | 16bit steps      |
| 59   | Nestelinjan lämpötila        | Celsius          |

## 🔧 ESP8266 + ESPHome -koodi

Toimiva YAML-konfiguraatio ESPHomelle löytyy kokonaisuudessaan lopusta. Käytetty palikka on D1 mini.  

Se sisältää:
- Uart-konfiguraation (`9600N1`)
- Sensoridefinitiot eri lämpötiloille, paineille ja statuksille
- Tilatekstien tulkinnan (esim. `0x80 = Käynnissä`, `0xA2 = Sulatus`)
- Filterin viallisille/epäluotettaville paketeille
- Pienen päivitysharvennuksen (esim. 1/3 paketista otetaan)

## 🛠️ Rajoitukset

- Koskee vain M-sarjan laitetta, ei toimi S-sarjassa
- Ei tarjoa ohjaustoimintoja, vain valvontaa
- Vaatii system controller -kortin ehkä (H-Link ei aktivoidu ilman sitä)
- Tunnisteet ja tavurakenne on saatu empiirisesti — ei virallista tukea

## 💬 Kiitokset ja lisenssi

Tämä projekti on koottu tekoälyn avulla kenttähavaintojen pohjalta **Jussin** toimesta — kiitos jakamisesta ja reverse engineering -innostasi!  
Julkaistu yhteisön hyväksi ilman takuuta, käytä omalla vastuulla. Parannusehdotuksia, havainnointeja ja täydennyksiä otetaan vastaan.

Esp yaml Home assistantille heti siitä perustietojen jälkeen alkaen "uart" kohdasta:

```yaml
uart:
  id: uart_bus
  rx_pin:
    number: GPIO3
    inverted: false
  tx_pin:
    number: GPIO1
    inverted: true
  baud_rate: 9600
  stop_bits: 1
  parity: NONE
  rx_buffer_size: 2048

globals:
  - id: packet_counter
    type: int
    restore_value: no
    initial_value: '0'

sensor:
  - platform: template
    name: "Lämmityksen pyynti"
    id: heating_setpoint
    unit_of_measurement: "°C"
    accuracy_decimals: 0
  - platform: template
    name: "Kiertovesi sisään"
    id: water_in
    unit_of_measurement: "°C"
    accuracy_decimals: 0
  - platform: template
    name: "Kiertovesi ulos"
    id: water_out
    unit_of_measurement: "°C"
    accuracy_decimals: 0
  - platform: template
    name: "Ulkolämpötila"
    id: outdoor_temp
    unit_of_measurement: "°C"
    accuracy_decimals: 0
  - platform: template
    name: "Kompressorin taajuus"
    id: compressor_freq
    unit_of_measurement: "Hz"
    accuracy_decimals: 0
  - platform: template
    name: "Imupaine"
    id: suction_pressure
    unit_of_measurement: "Bar"
    accuracy_decimals: 1
  - platform: template
    name: "Imulinjan lämpötila"
    id: suction_temp
    unit_of_measurement: "°C"
    accuracy_decimals: 0
  - platform: template
    name: "EEV avautuma"
    id: eev_steps
    unit_of_measurement: "steps"
    accuracy_decimals: 0
  - platform: template
    name: "Nestelinjan lämpötila"
    id: liquid_temp
    unit_of_measurement: "°C"
    accuracy_decimals: 0
    #Teen tähän nyt arvauksen kuumakaasun lämmölle, se vois olla tavu 55. Sille oma kohta tuolla lambdassa, ja kommenttikin tavuista.
    #Ei vaan voi kokeilla, kun ei voi laittaa virtoja Yutakiin. Nyt on eev auki tyhjäystä varten.
  - platform: template
    name: "Kuumakaasun lämpötila"
    id: discharge_temp
    unit_of_measurement: "°C"
    accuracy_decimals: 0
  - platform: template
    name: "Korkeapaine"
    id: discharge_pressure
    unit_of_measurement: "Bar"
    accuracy_decimals: 1
  - platform: template
    name: "joku1"
    id: joku1
    unit_of_measurement: "°C"
    accuracy_decimals: 0
  - platform: template
    name: "joku2"
    id: joku2
    unit_of_measurement: "°C"
    accuracy_decimals: 0
text_sensor:
  - platform: template
    name: "Yutaki tila"
    id: yutaki_state

interval:
  - interval: 200ms
    then:
      lambda: |-
        const int PACKET_SIZE = 61;
        while (id(uart_bus).available() >= PACKET_SIZE) {
          uint8_t header[3];
          id(uart_bus).read_array(header, 3);
          
          if (header[0] == 0x3F && header[1] == 0x01 && header[2] == 0x09) {
            uint8_t data[PACKET_SIZE - 3];
            id(uart_bus).read_array(data, PACKET_SIZE - 3);

            // Tarkistetaan ulkolämpötila ja kompuran taajuus ennen purkua
            int outdoor = (data[36] >= 0x80) ? (int)data[36] - 256 : (int)data[36];
            int compressor = data[45];

            if ((outdoor > -40 && outdoor < 50) && (compressor <= 120)) {
              id(packet_counter) += 1;
              if (id(packet_counter) >= 3) {
                id(packet_counter) = 0;
                
                auto read_temp = [](uint8_t val) -> int {
                  if (val >= 0x80) { return (int)val - 256; } else { return val; }
                };

                id(heating_setpoint).publish_state(read_temp(data[18]));  // tavu 22

                std::string state_txt;
                switch (data[30]) { // tavu 34
                  case 0x00: state_txt = "Sammutus"; break;
                  case 0x86: state_txt = "Kompuran odotus"; break;
                  case 0x80: state_txt = "Käynnissä"; break;
                  case 0x82: state_txt = "Tavoite saavutettu"; break;
                  case 0xA2: state_txt = "Sulatus"; break;
                  case 0x20: state_txt = "menovesianturi rikki"; break;
                  case 0x3F: state_txt = "vikatila?"; break;
                  case 0xF3: state_txt = "vikatila?"; break;
                  default: {
                    char buf[10];
                    sprintf(buf, "Tila %02X", data[30]);
                    state_txt = buf;
                  }
                }
                id(yutaki_state).publish_state(state_txt);

                id(joku1).publish_state((float)data[31]);  // tavu 35 Tämä nyt testiin, mikä voisi olla.
                id(joku2).publish_state((float)data[32]);  // tavu 36 Tämäkin testiin.
                id(water_in).publish_state(read_temp(data[34]));  // tavu 38
                id(water_out).publish_state(read_temp(data[35])); // tavu 39
                id(outdoor_temp).publish_state(outdoor); // tavu 40
                id(compressor_freq).publish_state(compressor); // tavu 49
                id(suction_pressure).publish_state((float)data[48] / 10.0f); // tavu 52, Pascal arvo tarvii jakaa kympillä
                id(discharge_pressure).publish_state((float)data[49] / 5.0f); // tavu 53, tämä kivasti puolitettu dataväylään, jako vaan vitosella.
                id(discharge_temp).publish_state(read_temp(data[51])); // Tavu 55. Tää tekoälyn teos näköjään vähentää header-tavut laskusta.
                id(suction_temp).publish_state(read_temp(data[52])); // tavu 56

                uint16_t eev = (data[53] << 8) | data[54]; // tavut 57 ja 58
                id(eev_steps).publish_state(eev);

                id(liquid_temp).publish_state(read_temp(data[55])); // tavu 59
              }
            } else {
              // Jos lämpötila tai kompuran taajuus epäilyttävä, hylätään paketti
              ESP_LOGW("yutaki_uart", "Hylätty paketti: ulkolämpö %d, kompuran taajuus %d", outdoor, compressor);
            }
          } else {
            // Ei oikea alku, pudotetaan 1 tavu ja jatketaan
            uint8_t dummy;
            id(uart_bus).read_byte(&dummy);
          }
        }




captive_portal:
```
