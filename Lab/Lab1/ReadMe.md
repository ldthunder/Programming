<p align="center">МИНИСТЕРСТВО НАУКИ  И ВЫСШЕГО ОБРАЗОВАНИЯ РОССИЙСКОЙ ФЕДЕРАЦИИ<br>
Федеральное государственное автономное образовательное учреждение высшего образования<br>
"КРЫМСКИЙ ФЕДЕРАЛЬНЫЙ УНИВЕРСИТЕТ им. В. И. ВЕРНАДСКОГО"<br>
ФИЗИКО-ТЕХНИЧЕСКИЙ ИНСТИТУТ<br>
Кафедра компьютерной инженерии и моделирования</p>
<br>
<h3 align="center">Отчёт по лабораторной работе № 1<br> по дисциплине "Программирование"</h3>
<br><br>
<p>студента 1 курса группы ИВТ-б-о-202(2)<br>
Бекбаев Ийстэел Азизович<br>
направления подготовки 09.03.01 "Информатика вычислительная техника"</p>
<br><br>
<table>
<tr><td>Научный руководитель<br> старший преподаватель кафедры<br> компьютерной инженерии и моделирования</td>
<td>(оценка)</td>
<td>Чабанов В.В.</td>
</tr>
</table>
<br><br>
<p align="center">Симферополь, 2021</p>
<hr>

## Цель работы:

1. Закрепить навыки разработки многофайловыx приложений.
2. Изучить способы работы с API web-сервиса.
3. Изучить процесс сериализации/десериализации данных в/из json.
4. Получить базовое представление о сетевом взаимодействии приложений.

## Постановка задачи

Разработать сервис предоставляющий данные о погоде в городе Симферополе. Источник данных о погоде: http://openweathermap.org/. В состав сервиса входит: серверное приложение на языке С++ и клиентское приложение на языке Python.

Серверное приложение предназначено для обслуживания клиентских приложений и сокращение количества запросов к сервису openweathermap.org. Сервер должен обеспечивать получение данных в формате JSON и виде html виджета.

Клиентское приложение должно иметь графический интерфейс отображающий сведения о погоде и возможность обновления данных по требованию пользователя.

## Выполнение работы

Для начала, получил все необходимые запросы, к которым мы будем обращаться на сервера openweathermap.org и worldtimeapi.org. Для этого:
1. Зарегистрировался на сайте http://openweathermap.org/ . В разделе My API Keys получил свой API ключ для получения данных с сайта: **063378bb49ad193cafdbeafe0b5819fd**
2. Далее, составил запрос для получения прогноза погоды: http://api.openweathermap.org//data/2.5/onecall?lat=44.952116&lon=34.102411&lang=ru&units=metric&appid=063378bb49ad193cafdbeafe0b5819fd
При вводе данного запроса в строку браузера получил json-ответ с нужной нам информацией о погоде (Рис. 1)

<p align="center"> <img src="./image/json.PNG"> </p>

<p align="center">Рис. 1 - Ответ о погоде в формате json<br>

3. Составил запрос для получения времени в Симферополе: http://worldtimeapi.org/api/timezone/Europe/Simferopol
Также, установил расширение для браузера, для отображения json - JsonDiscovery
При вводе данного запроса получил json ответ с нужной информацией о времени (Рис. 2)

<p align="center"> <img src="./image/time.PNG"> </p>

<p align="center">Рис. 2 - Ответ о времени<br>

4. Код серверного приложения на языке С++ (Рис. 3):

```C++
#include <iostream>
#include <string>
#include <iomanip>
#include <fstream>
#include <cpp_httplib/httplib.h>
#include <nlohmann/json.hpp>
using namespace std;
using json = nlohmann::json;
using namespace httplib;
json _json;
int unixtime = 0;
int cache_time_new = 0;
int temp = 0;
int w = 0;
json _json_cache_weth;
json _json_time;
void gen_response(const Request& req, Response& rez) {
	string str;
	ifstream l("Wid.html");                                                    
	Client Cli("http://worldtimeapi.org");										
	auto answer = Cli.Get("/api/timezone/Europe/Simferopol");						
	if (answer) {																	
		if (answer->status == 200) {											
			_json_time = json::parse(answer->body);								
			unixtime = _json_time["unixtime"].get<int>();
		}
		else {
			cout << "Status code: " << answer->status << endl;
		}
	}
	else {
		auto Error = answer.error();
		cout << "Error code: " << Error << std::endl;
	}
	Client cli("http://api.openweathermap.org");
	auto _answer = cli.Get("/data/2.5/onecall?lat=44.952116&lon=34.102411&lang=ru&units=metric&appid=063378bb49ad193cafdbeafe0b5819fd");
	if (_answer) {

		if (_answer->status == 200) {
			_json_cache_weth = json::parse(_answer->body);						
			cache_time_new = _json_cache_weth["hourly"][_json_cache_weth["hourly"].size() - 1]["dt"].get<int>(); 

			for (int i = _json_cache_weth["hourly"].size(); i > 0; --i) {
				for (int q = 0; q < _json_cache_weth["hourly"].size(); q++) {
					if (cache_time_new > unixtime and _json_cache_weth["hourly"][q]["dt"] <= cache_time_new) {
						temp = _json_cache_weth["hourly"][q]["dt"];												
						cache_time_new = temp;
						w = q;
					}
				}

			}

		}
		else {
			cout << "Status code: " << _answer->status << endl;
		}
	}
	else {
		auto Error = _answer.error();
		cout << "Error code: " << Error << endl;
	}
	_json["description"] = _json_cache_weth["hourly"][w]["weather"][0]["description"];
	_json["temp"] = to_string(_json_cache_weth["hourly"][w]["temp"].get<int>());
	getline(l, str, '\0');
	while (str.find("{hourly[i].temp}") != std::string::npos)
		str.replace(str.find("{hourly[i].temp}"), 16, std::to_string(_json_cache_weth["hourly"][w]["temp"].get<int>()));				
	str.replace(str.find("{hourly[i].weather[0].description}"), 34, _json_cache_weth["hourly"][w]["weather"][0]["description"]);				
	str.replace(str.find("{hourly[i].weather[0].icon}"), 27, _json_cache_weth["hourly"][w]["weather"][0]["icon"]);
	rez.set_content(str, "text/html");
}
void gen_response_raw(const Request& req, Response& rez) {
	Client("http://localhost:3000").Get("/");
	rez.set_content(_json.dump(), "text/json");
}
int main() {
	Server svr;
	svr.Get("/raw", gen_response_raw);
	svr.Get("/", gen_response);
	std::cout << "Start server... OK\n";
	svr.listen("localhost", 3000);
}

```

<p align="center"> <img  src="./image/localhost3000.PNG"> </p>

<p align="center">Рис. 3 - Виджет html<br>

5. Создал клиентское приложение с графическим интерфейсом на языке Python:



```Python
from tkinter import *
from tkinter.font import BOLD
import requests
import json

def reload_data(event=None):
	try:
		response = requests.get('http://localhost:3000/raw').content.decode("utf8")
		cache = json.loads(response)

		desc.config(text=str(cache["description"]))
		temp.config(text=str(cache["temp"]) + "°C")
	except requests.exceptions.ConnectionError:
		pass

root = Tk()
root.title("Погода")
root.pack_propagate()
root.bind("<Button-1>", reload_data)
top_frame =    Frame(root, bg="coral")
middle_frame = Frame(root, bg="ghost white")
bottom_frame = Frame(root, bg="coral", width=200, height=50)
top_frame.pack(side=TOP, fill=X)
middle_frame.pack(expand=True, fill=BOTH)
bottom_frame.pack(side=BOTTOM, fill=X)
city = Label(top_frame, font=("Times New Roman", 14), text="Симферополь", bg="coral")
desc = Label(top_frame, font=("Times New Roman", 14), bg="coral")
temp = Label(middle_frame, font=("Times New Roman", 48), bg="ghost white")
city.pack(pady=0)
desc.pack(pady=0)
temp.pack(expand=True)
root.mainloop()
```
6. Скриншот графического интерфейса клиентского приложения (Рис.4)
<p align="center"> <img  src="./image/widget.png"> </p>

<p align="center">Рис. 4 - Интерфейс клиентского приложения<br>

## Вывод:
Разработал серверное и клиентское приложение для которого была использована библиотека json и cpp-httplib. Научился работать с API. Разобрался в сетевом взаимодействии приложений и принципе работы Req запросов.