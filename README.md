package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strconv"
	"strings"
)

type RequestBody struct {
	Expression string `json:"expression"`
}

type ResponseBody struct {
	Result string `json:"result,omitempty"`
	Error  string `json:"error,omitempty"`
}

func calculate(expression string) (string, error) {
	result, err := eval(expression)
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%v", result), nil
}

func eval(expr string) (float64, error) {
	tokens := strings.Fields(expr)

	if len(tokens) != 3 {
		return 0, fmt.Errorf("недостаточно аргументов")
	}

	num1, err := strconv.ParseFloat(tokens[0], 64)
	if err != nil {
		return 0, fmt.Errorf("недопустимое число: %s", tokens[0])
	}

	operator := tokens[1]

	num2, err := strconv.ParseFloat(tokens[2], 64)
	if err != nil {
		return 0, fmt.Errorf("недопустимое число: %s", tokens[2])
	}

	switch operator {
	case "+":
		return num1 + num2, nil
	case "-":
		return num1 - num2, nil
	case "*":
		return num1 * num2, nil
	case "/":
		if num2 == 0 {
			return 0, fmt.Errorf("деление на ноль")
		}
		return num1 / num2, nil
	default:
		return 0, fmt.Errorf("неподдерживаемый оператор: %s", operator)
	}
}

func calculateHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "Метод не разрешен", http.StatusMethodNotAllowed)
		return
	}

	var requestBody RequestBody
	if err := json.NewDecoder(r.Body).Decode(&requestBody); err != nil || requestBody.Expression == "" {
		http.Error(w, "Недопустимое выражение", http.StatusUnprocessableEntity)
		return
	}

	result, err := calculate(requestBody.Expression)
	if err != nil {
		http.Error(w, "Внутренняя ошибка сервера", http.StatusInternalServerError)
		return
	}

	responseBody := ResponseBody{Result: result}
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(responseBody)
}

func main() {
	http.HandleFunc("/api/v1/calculate", calculateHandler)

	fmt.Println("Сервер запущен на порту 8080")
	if err := http.ListenAndServe(":8080", nil); err != nil {
		fmt.Println("Ошибка при запуске сервера:", err)
	}
}
