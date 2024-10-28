#include <iostream>
#include <string>
#include <unordered_map>
#include <sstream>
#include <curl/curl.h>
#include <json/json.h>

size_t WriteCallback(void* contents, size_t size, size_t nmemb, void* userp) {
    ((std::string*)userp)->append((char*)contents, size * nmemb);
    return size * nmemb;
}

std::unordered_map<std::string, double> getExchangeRates() {
    std::unordered_map<std::string, double> rates;
    CURL* curl;
    CURLcode res;
    std::string readBuffer;

    curl = curl_easy_init();
    if (curl) {
        curl_easy_setopt(curl, CURLOPT_URL, "https://api.exchangerate-api.com/v4/latest/USD"); // Замените на ваш API
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &readBuffer);
        res = curl_easy_perform(curl);
        curl_easy_cleanup(curl);
    }

    // Проверка на наличие данных
    if (res != CURLE_OK) {
        std::cerr << "Ошибка запроса: " << curl_easy_strerror(res) << std::endl;
        return rates; // Возвращаем пустую карту, если произошла ошибка
    }

    Json::Value jsonData;
    Json::CharReaderBuilder readerBuilder;
    std::string errors;
    std::istringstream s(readBuffer);

    if (Json::parseFromStream(readerBuilder, s, &jsonData, &errors)) {
        if (jsonData.isMember("rates")) {
            for (const auto& currency : jsonData["rates"].getMemberNames()) {
                rates[currency] = jsonData["rates"][currency].asDouble();
            }
        }
        else {
            std::cerr << "Ошибка: нет данных о курсах валют." << std::endl;
        }
    }
    else {
        std::cerr << "Ошибка парсинга JSON: " << errors << std::endl;
    }
    return rates;
}

double convertCurrency(double amount, const std::string& from, const std::string& to, const std::unordered_map<std::string, double>& rates) {
    auto itFrom = rates.find(from);
    auto itTo = rates.find(to);

    if (itFrom == rates.end()) {
        throw std::invalid_argument("Неверная валюта 'from': " + from);
    }
    if (itTo == rates.end()) {
        throw std::invalid_argument("Неверная валюта 'to': " + to);
    }
    return (amount / itFrom->second) * itTo->second; // Конвертация валюты
}
void displayMenu(const std::unordered_map<std::string, double>& rates, std::string& fromCurrency, std::string& toCurrency) {
    if (rates.empty()) {
        std::cout << "Нет доступных валют для отображения." << std::endl;
        return;
    }

    std::cout << "Доступные валюты:\n";
    for (const auto& rate : rates) {
        std::cout << rate.first << "\n"; // Вывод всех доступных валют
    }

    std::cout << "Выберите валюту из которой хотите перевести:\n";
    std::cout << "Введите код валюты: ";
    std::cin >> fromCurrency; // Ввод начальной валюты

    std::cout << "Выберите валюту в которую хотите перевести:\n";
    std::cout << "Введите код валюты: ";
    std::cin >> toCurrency; // Ввод конечной валюты
}

int main() {
setlocale(LC_ALL, "ru");
    std::unordered_map<std::string, double> rates = getExchangeRates();

    // Проверяем, были ли успешно получены валютные курсы
    if (rates.empty()) {
        std::cerr << "Не удалось загрузить курсы валют." << std::endl;
        return 1; // Выходим с кодом ошибки
    }

    while (true) {
        std::string fromCurrency, toCurrency;
        displayMenu(rates, fromCurrency, toCurrency);  // Передаем обе переменные

        std::cout << "Введите сумму валюты или 'exit' для выхода: ";
        double amount;
        std::string input;

        std::getline(std::cin >> std::ws, input); // Чтение с удалением пробелов

        if (input == "exit") {
            break; // Выход из программы
        }

        try {
            amount = std::stod(input); // Преобразование введенной строки в число
            double result = convertCurrency(amount, fromCurrency, toCurrency, rates);

            std::cout << amount << " " << fromCurrency << " = " << result << " " << toCurrency << std::endl; // Вывод результата
        }
        catch (const std::exception& e) {
            std::cerr << e.what() << std::endl; // Вывод ошибки
        }
    }

    return 0;
}
