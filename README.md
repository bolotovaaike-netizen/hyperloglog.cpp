#include <iostream>
#include <vector>
#include <cmath>
#include <algorithm>
#include <unordered_set>
#include <cstdlib> // rand

class HyperLogLog {
private:
    int p;
    int m;
    std::vector<int> registers;
    double alpha_m;

    double getAlpha() {
        if (m == 16) return 0.673;
        if (m == 32) return 0.697;
        if (m == 64) return 0.709;
        return 0.7213 / (1 + 1.079 / m);
    }

    // Очень простой хеш (работает для учебных целей)
    unsigned long long hash(unsigned long long x) {
        x ^= (x >> 33);
        x *= 0xff51afd7ed558ccdULL;
        x ^= (x >> 33);
        x *= 0xc4ceb9fe1a85ec53ULL;
        x ^= (x >> 33);
        return x;
    }

    int leadingZeros(unsigned long long x) {
        int n = 1;
        while ((x & (1ULL << 63)) == 0 && n <= 64) {
            x <<= 1;
            n++;
        }
        return n;
    }

public:
    HyperLogLog(int bits = 14) : p(bits), m(1 << bits) {
        registers.resize(m, 0);
        alpha_m = getAlpha();
    }

    void add(unsigned long long value) {
        unsigned long long x = hash(value);
        int idx = x >> (64 - p);
        unsigned long long w = x << p;
        int rho = leadingZeros(w);
        if (rho > registers[idx]) registers[idx] = rho;
    }

    unsigned long long estimate() {
        double Z = 0;
        for (int v : registers) Z += pow(2.0, -v);
        double E = alpha_m * m * m / Z;

        int V = std::count(registers.begin(), registers.end(), 0);
        if (E <= 5.0 / 2 * m && V != 0) {
            E = m * log((double)m / V);
        } else if (E > (1.0 / 30.0) * pow(2,32)) {
            E = -(pow(2,32) * log(1 - E / pow(2,32)));
        }
        return (unsigned long long)E;
    }

    void merge(const HyperLogLog &other) {
        if (m != other.m) throw std::runtime_error("Количество корзин должно совпадать");
        for (int i = 0; i < m; i++) registers[i] = std::max(registers[i], other.registers[i]);
    }
};

int main() {
    HyperLogLog hll;

    std::unordered_set<unsigned long long> unique_values;
    for (int i = 0; i < 100000; i++) {
        unsigned long long x = rand() % 200000;
        unique_values.insert(x);
        hll.add(x);
    }

    std::cout << "Реальное количество уникальных: " << unique_values.size() << std::endl;
    std::cout << "Оценка HyperLogLog: " << hll.estimate() << std::endl;

    return 0;
}
