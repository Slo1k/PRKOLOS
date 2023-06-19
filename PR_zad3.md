# Mnożenie macierzy / zadanie dotyczące kodu

![zad3.1](./img_zad/zad3/zad3.1.png) \
*Zadanie 3.1*

---

## Mnożenie macierzy - jeden blok

1) kod
```cpp
void MatrixMulKernel_1(float* Ad, float* Bd, float* Cd, int WIDTH)
{ 
    int tx = threadIdx.x; //nr kolumny liczonego elementu 
    int ty = threadIdx.y; //nr wiersza liczonego elementuW

    int Row = ty;
    int Col = tx;

    float C_local = 0; 

    for (int k = 0; k < WIDTH; ++k) {
        // float A_d_element = Ad[ty * WIDTH + k]; //wszystkie elementy wiersza po kolei
        // float B_d_element = Bd[k * WIDTH + tx]; //wszystkie elementy kolumny po kolei
        // C_local += A_d_element * B_d_element;
        
        // C_local += Ad[Row * WIDTH + k] * Bd[k * WIDTH + Col];

        C_local += A_d[Row][k] * B_d[k][Col]; 
    }

    //zapis wyniku
    //Cd[Row * WIDTH + Col] = C_local; 
    Cd[Row][Col] = C_local;
}
```

2) postać wywołania kernela (parametry i konfiguracja)
```cpp

dim3 block(WIDTH, WIDTH);
dim3 grid(1, 1);

MatrixMulKernel_1<<< grid, block >>>(Ad,Bd,Cd,WIDTH); 

```

3) Współczynnik CGMA wskazuje na stosunek liczby operacji na danych do liczby umożliwiających te operacje dostępów do pamięci globalnej. W tym wypadku na 2 operacje (+,*) przypadają 2 dostępy do pamięci globalnej, czyli CGMA wynosi 2/2=1. Wartość wyrażona w bajtach: 2/8bajtów = 1/4 FLOP/S

4) ocena jakości rozwiązania
* A_d_element - zazwyczaj jeden transfer - jednakowa wartość dla wszystkich wątków wiązki (jednakowe ty i k), słabe wykorzystanie pojemności transferu
* B_d_element - transfer efektywny - dostępy są zazwyczaj łączone gdyż sąsiednie lokacje pamięci (kolejne tx) żądane przez 1/2 wiązki (wiązkę wątków)
* Cd - jakość dostępu jak do Bd

---

## Mnożenie macierzy - wiele bloków

1) kod
```cpp
void MatrixMulKernel_2(float* Ad, float* Bd, float* Cd, int WIDTH)
{
    // wyznaczenie indeksu wiersza/kolumny obliczanego elementu tablicy Cd
    int Row = blockId.y * blockDim.y + threadId.y; 
    int Col = blockId.x * blockDim.x + threadId.x; 
    float C_local = 0;
    // każdy wątek z bloku oblicza jeden element macierzy 
    for (int k = 0; k < WIDTH; ++k) {
        C_local += A_d[Row][k] * B_d[k][Col]; 
    }
    
    Cd[Row][Col] = C_local; // zapis wyniku 
} 
```

2) postać wywołania kernela (parametry i konfiguracja)
```cpp

int tmp = sufit(WIDTH / SUB_WIDTH);

dim3 block(SUB_WIDTH, SUB_WIDTH); 
dim3 grid(tmp, tmp);

MatrixMulKernel_2<<< grid, block >>>(Ad, Bd, Cd,WIDTH); 

```

3)  Współczynnik CGMA wskazuje na stosunek liczby operacji na danych do liczby umożliwiających te operacje dostępów do pamięci globalnej. W tym wypadku na 2 operacje (+,*) przypadają 2 dostępy do pamięci globalnej, czyli CGMA wynosi 2/2=1. Wartość wyrażona w bajtach: 2/8bajtów = 1/4 FLOP/S

4) ocena jakości rozwiązania
* A_d_element - zazwyczaj jeden transfer - jednakowa wartość dla wszystkich wątków wiązki (jednakowe ty i k), słabe wykorzystanie pojemności transferu
* B_d_element - transfer efektywny - dostępy są zazwyczaj łączone gdyż sąsiednie lokacje pamięci (kolejne tx) żądane przez 1/2 wiązki (wiązkę wątków)
* Cd - jakość dostępu jak do Bd

---

## Mnożenie macierzy - pamięć współdzielona

1) kod
```cpp
void MatrixMulKernel_3(float* Ad, float* Bd, float* Cd, int Width) 
{
    __shared__float Ads[SUB_WIDTH][SUB_WIDTH];
    __shared__float Bds[SUB_WIDTH][SUB _WIDTH];
    // określenie obliczanego przez wątek elementu macierzy (jak w poprzednim kodzie)
    //tx, ty to identyfikatory wątków w ramach bloku, Row i Col - analogicznie 
    for (int m = 0; m < WIDTH/ SUB_WIDTH; ++m) { 
        Ads[ty][tx] = Ad [Row][m*SUB_WIDTH + tx]; 
        //kolejny element dla sąsiedniego wątku
        Bds[ty][tx] = Bd[m*SUB_WIDTH + ty][Col]; 
        // używana kolumna – jakość pobrań ?
        __syncthreads(); 
        for (int k = 0; k < SUB_WIDTH; ++k) 
        C_local += Ads[ty][k] * Bds[k][tx]; __syncthreads(); 
    }

    Cd[Row][Col] = C_local; // zapis wyniku 
}
```
> todo opis

## Wyznaczenie maksimum elementów z tablicy (PRAM)
![PlfkdoFlhv](img_zad\zad3\zad3_pram.png)

## Kod scalenia wektora
![](img_zad\zad3\zad3_scalanie.png)


---

## Long story short (warianty 1&2) 

1) kod
```cpp
void MatrixMulKernel_1_2(float* Ad, float* Bd, float* Cd, int WIDTH)
{ 
    // wariant dla pojedynczego bloku

    int v1a = threadIdx.x; //nr kolumny liczonego elementu 
    int v1b = threadIdx.y; //nr wiersza liczonego elementu

    //-----------------------------------------------------------------
    // wariant dla wielu bloków

    // wyznaczenie indeksu wiersza/kolumny obliczanego elementu tablicy Cd
    int v2a = blockId.y * blockDim.y + threadId.y; 
    int v2b = blockId.x * blockDim.x + threadId.x; 

    //-----------------------------------------------------------------
    //część wspólna dla każdego wariantu

    int Row = (...);
    int Col = (...);

    float C_local = 0; 

    for (int k = 0; k < WIDTH; ++k) {
        C_local += A_d[Row][k] * B_d[k][Col]; 
    }

    Cd[Row][Col] = C_local; //zapis wyniku
}
```