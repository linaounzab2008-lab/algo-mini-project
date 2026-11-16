# algo-mini-project
réaliser une application concrète : une Facture Unique de services (eau, gaz, électricité, téléphone et autres),

/* =====================================================================
   FACTUREUNIQUE
   Smart Multi-Utility Billing System
   ASDS Mini Project - ESTIN

   Single-file edition:
   bill.h + bill.c + main.c combined

   Features:
   1. Electricity, Gas, Water, Telephone and Internet, Waste collection
   2. Progressive tariff calculations
   3. Automatic VAT calculations
   4. Customer and invoice identification
   5. Professional invoice generation
   6. Consumption analysis
   7. Billing history
   8. Billing period comparison
   9. Payment status
   10. Discount management
   11. Invoice export to text files
   12. Robust input validation

   Compile:
   gcc -std=c11 -Wall -Wextra -Wpedantic FactureUnique.c -o FactureUnique

   ===================================================================== */


/* =====================================================================
   SECTION 1 - LIBRARIES AND CONSTANTS
   ===================================================================== */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <math.h>
#include <errno.h>
#include <ctype.h>


/* ---------------- Electricity tariffs (DA/kWh) ---------------- */

#define ELEC_LIMIT1       100.0
#define ELEC_LIMIT2       300.0
#define ELEC_PRICE1       2.00
#define ELEC_PRICE2       4.00
#define ELEC_PRICE3       6.00
#define ELEC_FIXED_FEE    100.0
#define ELEC_VAT          0.09


/* ---------------- Gas tariffs (DA/m3) ---------------- */

#define GAS_LIMIT1        100.0
#define GAS_LIMIT2        300.0
#define GAS_PRICE1        1.50
#define GAS_PRICE2        3.00
#define GAS_PRICE3        4.50
#define GAS_FIXED_FEE     50.0
#define GAS_VAT           0.09


/* ---------------- Water tariffs (DA/m3) ---------------- */

#define WATER_LIMIT1      20.0
#define WATER_LIMIT2      40.0
#define WATER_PRICE1      6.00
#define WATER_PRICE2      20.00
#define WATER_PRICE3      40.00
#define WATER_SANITATION  0.10
#define WATER_FIXED_FEE   100.0
#define WATER_VAT         0.09


/* ---------------- Telephone and Internet ---------------- */

#define PHONE_SUBSCRIPTION 600.0
#define PHONE_INCLUDED_MIN 100.0
#define PHONE_EXTRA_PRICE  3.00

#define NET_PRICE_4MB      1000.0
#define NET_PRICE_8MB      1500.0
#define NET_PRICE_20MB     2500.0

#define PHONE_VAT          0.19


/* ---------------- Waste collection ---------------- */

#define WASTE_FIXED_FEE    150.0
#define WASTE_VAT          0.09


/* ---------------- General settings ---------------- */

#define MAX_LINES          5
#define MAX_HISTORY        100
#define STR_LEN            50
#define INPUT_LEN          150

#define S_ELEC             1
#define S_GAS              2
#define S_WATER            3
#define S_PHONE            4
#define S_OTHER            5


/* =====================================================================
   SECTION 2 - DATA STRUCTURES
   ===================================================================== */


/* One service on the invoice */

typedef struct {

    char name[STR_LEN];

    int has_index;

    double previous;
    double current;
    double consumption;

    double before_tax;
    double vat;
    double total;

} BillLine;


/* Complete customer invoice */

typedef struct {

    char invoice_id[STR_LEN];

    char customer[STR_LEN];
    char account[STR_LEN];
    char period[STR_LEN];

    BillLine lines[MAX_LINES];

    int count;

    double total_before_tax;
    double total_vat;

    double discount_rate;
    double discount_amount;

    double total_before_discount;
    double total_to_pay;

    int is_paid;

} Bill;


/* Billing history */

typedef struct {

    Bill bills[MAX_HISTORY];

    int count;

} BillHistory;


/* =====================================================================
   SECTION 3 - FUNCTION PROTOTYPES
   ===================================================================== */


/* Calculations */

double progressive_amount(double cons,
                          double lim1,
                          double lim2,
                          double p1,
                          double p2,
                          double p3);

double electricity_amount(double kwh);
double gas_amount(double m3);
double water_amount(double m3);
double phone_amount(double minutes);
double internet_amount(int mbps);

double compute_vat(double amount, double rate);


/* Bill management */

void bill_init(Bill *b);

int bill_add_line(Bill *b,
                  const char *name,
                  int has_index,
                  double prev,
                  double cur,
                  double before_tax,
                  double vat_rate);

void bill_compute_totals(Bill *b);

void generate_invoice_id(Bill *b);

void display_bill(const Bill *b, FILE *out);


/* Input validation */

static int read_line(char *buf, int size);

double read_double(const char *prompt, double min);

int read_int(const char *prompt, int min, int max);

void read_string(const char *prompt, char *buf, int size);

int read_yes_no(const char *prompt);

double read_percentage(const char *prompt);


/* Service functions */

static void read_indexes(const char *unit,
                         double *prev,
                         double *cur);

static void add_electricity(Bill *b);
static void add_gas(Bill *b);
static void add_water(Bill *b);
static void add_phone(Bill *b);
static void add_other(Bill *b);


/* Additional features */

void analyze_consumption(const Bill *b);

void history_init(BillHistory *h);

void history_add(BillHistory *h, const Bill *b);

void display_history(const BillHistory *h);

void compare_previous_bill(const BillHistory *h,
                           const Bill *current);


/* Invoice management */

void save_bill(const Bill *b);

void print_menu(const int added[]);


/* =====================================================================
   SECTION 4 - TARIFF CALCULATIONS
   ===================================================================== */


/* Generic progressive tariff calculation */

double progressive_amount(double cons,
                          double lim1,
                          double lim2,
                          double p1,
                          double p2,
                          double p3)
{
    double amount = 0.0;

    if (cons <= lim1) {

        amount = cons * p1;

    }
    else if (cons <= lim2) {

        amount = lim1 * p1
               + (cons - lim1) * p2;

    }
    else {

        amount = lim1 * p1
               + (lim2 - lim1) * p2
               + (cons - lim2) * p3;

    }

    return amount;
}


/* Electricity calculation */

double electricity_amount(double kwh)
{
    return progressive_amount(
        kwh,
        ELEC_LIMIT1,
        ELEC_LIMIT2,
        ELEC_PRICE1,
        ELEC_PRICE2,
        ELEC_PRICE3
    ) + ELEC_FIXED_FEE;
}


/* Gas calculation */

double gas_amount(double m3)
{
    return progressive_amount(
        m3,
        GAS_LIMIT1,
        GAS_LIMIT2,
        GAS_PRICE1,
        GAS_PRICE2,
        GAS_PRICE3
    ) + GAS_FIXED_FEE;
}


/* Water calculation */

double water_amount(double m3)
{
    double base;

    double sanitation;

    base = progressive_amount(
        m3,
        WATER_LIMIT1,
        WATER_LIMIT2,
        WATER_PRICE1,
        WATER_PRICE2,
        WATER_PRICE3
    );

    sanitation = base * WATER_SANITATION;

    return base + sanitation + WATER_FIXED_FEE;
}


/* Telephone calculation */

double phone_amount(double minutes)
{
    double amount = PHONE_SUBSCRIPTION;

    if (minutes > PHONE_INCLUDED_MIN) {

        amount += (minutes - PHONE_INCLUDED_MIN)
                  * PHONE_EXTRA_PRICE;

    }

    return amount;
}


/* Internet package calculation */

double internet_amount(int mbps)
{
    switch (mbps) {

        case 0:
            return 0.0;

        case 4:
            return NET_PRICE_4MB;

        case 8:
            return NET_PRICE_8MB;

        case 20:
            return NET_PRICE_20MB;

        default:
            return -1.0;

    }
}


/* VAT calculation */

double compute_vat(double amount, double rate)
{
    return amount * rate;
}


/* =====================================================================
   SECTION 5 - BILL MANAGEMENT
   ===================================================================== */


/* Initialize an empty invoice */

void bill_init(Bill *b)
{
    memset(b, 0, sizeof(Bill));

    b->is_paid = 0;
}


/* Generate a unique invoice identifier */

void generate_invoice_id(Bill *b)
{
    static unsigned int counter = 0;

    time_t now = time(NULL);

    counter++;

    snprintf(
        b->invoice_id,
        sizeof(b->invoice_id),
        "FU-%lld-%03u",
        (long long)now,
        counter
    );
}


/* Add a service to the invoice */

int bill_add_line(Bill *b,
                  const char *name,
                  int has_index,
                  double prev,
                  double cur,
                  double before_tax,
                  double vat_rate)
{
    BillLine *l;

    if (b == NULL || name == NULL)
        return 0;

    if (b->count >= MAX_LINES)
        return 0;

    if (!isfinite(before_tax) ||
        !isfinite(vat_rate) ||
        before_tax < 0.0 ||
        vat_rate < 0.0)
        return 0;

    l = &b->lines[b->count];

    snprintf(l->name, sizeof(l->name), "%s", name);

    l->has_index = has_index;

    l->previous = prev;
    l->current = cur;

    l->consumption = has_index ? cur - prev : 0.0;

    l->before_tax = before_tax;

    l->vat = compute_vat(before_tax, vat_rate);

    l->total = l->before_tax + l->vat;

    b->count++;

    return 1;
}


/* Calculate invoice totals */

void bill_compute_totals(Bill *b)
{
    int i;

    b->total_before_tax = 0.0;

    b->total_vat = 0.0;

    b->total_before_discount = 0.0;

    b->discount_amount = 0.0;

    b->total_to_pay = 0.0;

    for (i = 0; i < b->count; i++) {

        b->total_before_tax += b->lines[i].before_tax;

        b->total_vat += b->lines[i].vat;

    }

    b->total_before_discount =
        b->total_before_tax + b->total_vat;

    b->discount_amount =
        b->total_before_discount * b->discount_rate / 100.0;

    b->total_to_pay =
        b->total_before_discount - b->discount_amount;
}


/* =====================================================================
   SECTION 6 - INPUT VALIDATION
   ===================================================================== */


/* Safely read one complete line */

static int read_line(char *buf, int size)
{
    size_t len;

    int ch;

    if (fgets(buf, size, stdin) == NULL) {

        printf("\nEnd of input. Goodbye.\n");

        exit(0);

    }

    len = strlen(buf);

    if (len > 0 && buf[len - 1] == '\n') {

        buf[len - 1] = '\0';

        if (len > 1 && buf[len - 2] == '\r')
            buf[len - 2] = '\0';

    }
    else if (!feof(stdin)) {

        while ((ch = getchar()) != '\n' && ch != EOF)
            ;

        printf("  [!] Input is too long. Try again.\n");

        return 0;

    }

    return 1;
}


/* Read a valid double */

double read_double(const char *prompt, double min)
{
    char buf[INPUT_LEN];

    char *end;

    double value;

    for (;;) {

        printf("%s", prompt);

        if (!read_line(buf, sizeof(buf)))
            continue;

        errno = 0;

        value = strtod(buf, &end);

        while (isspace((unsigned char)*end))
            end++;

        if (end == buf || *end != '\0' ||
            errno == ERANGE || !isfinite(value)) {

            printf("  [!] Invalid number. Try again.\n");

        }
        else if (value < min) {

            printf("  [!] Value must be >= %.2f.\n", min);

        }
        else {

            return value;

        }

    }
}


/* Read a valid integer */

int read_int(const char *prompt, int min, int max)
{
    char buf[INPUT_LEN];

    char *end;

    long value;

    for (;;) {

        printf("%s", prompt);

        if (!read_line(buf, sizeof(buf)))
            continue;

        errno = 0;

        value = strtol(buf, &end, 10);

        while (isspace((unsigned char)*end))
            end++;

        if (end == buf || *end != '\0' ||
            errno == ERANGE) {

            printf("  [!] Invalid integer. Try again.\n");

        }
        else if (value < min || value > max) {

            printf(
                "  [!] Enter a value between %d and %d.\n",
                min,
                max
            );

        }
        else {

            return (int)value;

        }

    }
}


/* Read a non-empty string */

void read_string(const char *prompt, char *buf, int size)
{
    for (;;) {

        printf("%s", prompt);

        if (!read_line(buf, size))
            continue;

        if (buf[0] != '\0')
            return;

        printf("  [!] This field cannot be empty.\n");

    }
}


/* Read yes/no answer */

int read_yes_no(const char *prompt)
{
    char buf[INPUT_LEN];

    char *p;

    for (;;) {

        printf("%s (y/n): ", prompt);

        if (!read_line(buf, sizeof(buf)))
            continue;

        p = buf;

        while (isspace((unsigned char)*p))
            p++;

        if ((p[0] == 'y' || p[0] == 'Y') &&
            p[1] == '\0')
            return 1;

        if ((p[0] == 'n' || p[0] == 'N') &&
            p[1] == '\0')
            return 0;

        printf("  [!] Please enter y or n.\n");

    }
}


/* Read a discount percentage */

double read_percentage(const char *prompt)
{
    double value;

    do {

        value = read_double(prompt, 0.0);

        if (value > 100.0)
            printf("  [!] Discount cannot exceed 100%%.\n");

    } while (value > 100.0);

    return value;
}


/* =====================================================================
   SECTION 7 - SERVICE INPUT FUNCTIONS
   ===================================================================== */


/* Read previous and current indexes */

static void read_indexes(const char *unit,
                         double *prev,
                         double *cur)
{
    *prev = read_double("  Previous index: ", 0.0);

    do {

        *cur = read_double("  Current index : ", 0.0);

        if (*cur < *prev) {

            printf(
                "  [!] Current index cannot be lower than previous index (%s).\n",
                unit
            );

        }

    } while (*cur < *prev);
}


/* Electricity service */

static void add_electricity(Bill *b)
{
    double prev, cur;

    printf("\n========== ELECTRICITY ==========\n");

    read_indexes("kWh", &prev, &cur);

    bill_add_line(
        b,
        "Electricity",
        1,
        prev,
        cur,
        electricity_amount(cur - prev),
        ELEC_VAT
    );
}


/* Gas service */

static void add_gas(Bill *b)
{
    double prev, cur;

    printf("\n========== GAS ==========\n");

    read_indexes("m3", &prev, &cur);

    bill_add_line(
        b,
        "Gas",
        1,
        prev,
        cur,
        gas_amount(cur - prev),
        GAS_VAT
    );
}


/* Water service */

static void add_water(Bill *b)
{
    double prev, cur;

    printf("\n========== WATER ==========\n");

    read_indexes("m3", &prev, &cur);

    bill_add_line(
        b,
        "Water",
        1,
        prev,
        cur,
        water_amount(cur - prev),
        WATER_VAT
    );
}


/* Telephone and Internet service */

static void add_phone(Bill *b)
{
    double prev, cur;

    double amount, net;

    int mbps;

    printf("\n====== TELEPHONE & INTERNET ======\n");

    read_indexes("minutes", &prev, &cur);

    amount = phone_amount(cur - prev);

    printf("\nAvailable Internet packages:\n");

    printf("  0 Mbps  : No Internet\n");
    printf("  4 Mbps  : %.2f DA\n", NET_PRICE_4MB);
    printf("  8 Mbps  : %.2f DA\n", NET_PRICE_8MB);
    printf(" 20 Mbps  : %.2f DA\n", NET_PRICE_20MB);

    do {

        mbps = read_int(
            "Choose your package: ",
            0,
            20
        );

        net = internet_amount(mbps);

        if (net < 0)
            printf("  [!] Choose 0, 4, 8 or 20.\n");

    } while (net < 0);

    bill_add_line(
        b,
        "Telephone + Internet",
        1,
        prev,
        cur,
        amount + net,
        PHONE_VAT
    );
}


/* Waste collection service */

static void add_other(Bill *b)
{
    printf("\n========== WASTE COLLECTION ==========\n");

    printf(
        "Fixed fee: %.2f DA\n",
        WASTE_FIXED_FEE
    );

    bill_add_line(
        b,
        "Waste collection",
        0,
        0,
        0,
        WASTE_FIXED_FEE,
        WASTE_VAT
    );
}


/* =====================================================================
   SECTION 8 - CONSUMPTION ANALYSIS
   ===================================================================== */


/* Analyze consumption according to tariff thresholds */

void analyze_consumption(const Bill *b)
{
    int i;

    double limit1, limit2;

    printf("\n");
    printf("============================================================\n");
    printf("                 SMART CONSUMPTION ANALYSIS                 \n");
    printf("============================================================\n");

    for (i = 0; i < b->count; i++) {

        const BillLine *l = &b->lines[i];

        if (!l->has_index)
            continue;

        if (strcmp(l->name, "Electricity") == 0) {

            limit1 = ELEC_LIMIT1;
            limit2 = ELEC_LIMIT2;

        }
        else if (strcmp(l->name, "Gas") == 0) {

            limit1 = GAS_LIMIT1;
            limit2 = GAS_LIMIT2;

        }
        else if (strcmp(l->name, "Water") == 0) {

            limit1 = WATER_LIMIT1;
            limit2 = WATER_LIMIT2;

        }
        else {

            limit1 = 100.0;
            limit2 = 300.0;

        }

        printf("\nService: %s\n", l->name);

        printf(
            "Consumption: %.2f units\n",
            l->consumption
        );

        if (l->consumption <= limit1) {

            printf("Consumption band: LOW\n");

            printf(
                "Recommendation: Your consumption is within the first tariff tier.\n"
            );

        }
        else if (l->consumption <= limit2) {

            printf("Consumption band: MODERATE\n");

            printf(
                "Recommendation: Monitor your usage to avoid higher tariff rates.\n"
            );

        }
        else {

            printf("Consumption band: HIGH\n");

            printf(
                "Recommendation: Consider reducing unnecessary consumption.\n"
            );

        }

    }

    printf("\n============================================================\n");
    printf("Note: Classification is based on tariff thresholds.\n");
    printf("============================================================\n");
}


/* =====================================================================
   SECTION 9 - BILLING HISTORY
   ===================================================================== */


/* Initialize history */

void history_init(BillHistory *h)
{
    memset(h, 0, sizeof(BillHistory));
}


/* Add an invoice to history */

void history_add(BillHistory *h, const Bill *b)
{
    if (h->count < MAX_HISTORY) {

        h->bills[h->count] = *b;

        h->count++;

    }
    else {

        memmove(
            &h->bills[0],
            &h->bills[1],
            sizeof(Bill) * (MAX_HISTORY - 1)
        );

        h->bills[MAX_HISTORY - 1] = *b;

    }
}


/* Display billing history */

void display_history(const BillHistory *h)
{
    int i;

    if (h->count == 0) {

        printf("\nNo previous invoices available.\n");

        return;

    }

    printf("\n");
    printf("============================================================\n");
    printf("                       BILLING HISTORY                      \n");
    printf("============================================================\n");

    printf(
        "%-20s %-20s %-12s %-12s\n",
        "Invoice ID",
        "Customer",
        "Amount (DA)",
        "Status"
    );

    printf("------------------------------------------------------------\n");

    for (i = 0; i < h->count; i++) {

        const Bill *b = &h->bills[i];

        printf(
            "%-20s %-20.20s %12.2f %-12s\n",
            b->invoice_id,
            b->customer,
            b->total_to_pay,
            b->is_paid ? "PAID" : "UNPAID"
        );

    }

    printf("============================================================\n");
}


/* Compare current invoice with the previous invoice
   belonging to the same account. */

void compare_previous_bill(const BillHistory *h,
                           const Bill *current)
{
    int i, j;

    const Bill *previous = NULL;

    for (i = h->count - 1; i >= 0; i--) {

        if (strcmp(
            h->bills[i].account,
            current->account
        ) == 0) {

            previous = &h->bills[i];

            break;

        }

    }

    if (previous == NULL) {

        printf("\nNo previous bill found for this account.\n");

        return;

    }

    printf("\n");
    printf("============================================================\n");
    printf("                   BILLING PERIOD COMPARISON                \n");
    printf("============================================================\n");

    printf("Previous invoice: %s\n", previous->invoice_id);

    printf("Current invoice : %s\n", current->invoice_id);

    for (i = 0; i < current->count; i++) {

        const BillLine *now = &current->lines[i];

        for (j = 0; j < previous->count; j++) {

            const BillLine *old = &previous->lines[j];

            if (strcmp(now->name, old->name) == 0 &&
                now->has_index && old->has_index) {

                double difference =
                    now->consumption - old->consumption;

                printf("\n%s\n", now->name);

                printf(
                    "Previous: %.2f\n",
                    old->consumption
                );

                printf(
                    "Current : %.2f\n",
                    now->consumption
                );

                if (difference > 0.0) {

                    printf(
                        "Change  : +%.2f units (increased)\n",
                        difference
                    );

                }
                else if (difference < 0.0) {

                    printf(
                        "Change  : %.2f units (decreased)\n",
                        difference
                    );

                }
                else {

                    printf("Change  : No change\n");

                }

                break;

            }

        }

    }

    printf("\n============================================================\n");
}


/* =====================================================================
   SECTION 10 - PROFESSIONAL INVOICE DISPLAY
   ===================================================================== */


void display_bill(const Bill *b, FILE *out)
{
    int i;

    const char *sep =
        "+----------------------+----------+----------+"
        "-------------+-------------+------------+-------------+\n";

    fprintf(out, "\n");

    fprintf(
        out,
        "============================================================\n"
    );

    fprintf(
        out,
        "                     FACTUREUNIQUE                          \n"
    );

    fprintf(
        out,
        "              UNIFIED UTILITY BILLING SYSTEM                \n"
    );

    fprintf(
        out,
        "============================================================\n"
    );

    fprintf(out, "Invoice ID     : %s\n", b->invoice_id);

    fprintf(out, "Customer       : %s\n", b->customer);

    fprintf(out, "Account number : %s\n", b->account);

    fprintf(out, "Billing period : %s\n", b->period);

    fprintf(
        out,
        "Payment status : %s\n",
        b->is_paid ? "PAID" : "UNPAID"
    );

    fprintf(out, "\n");

    fprintf(out, "%s", sep);

    fprintf(
        out,
        "| %-20s | %8s | %8s | %11s | %11s | %10s | %11s |\n",
        "Service",
        "Previous",
        "Current",
        "Consumption",
        "Before tax",
        "VAT",
        "Total (DA)"
    );

    fprintf(out, "%s", sep);

    for (i = 0; i < b->count; i++) {

        const BillLine *l = &b->lines[i];

        if (l->has_index) {

            fprintf(
                out,
                "| %-20.20s | %8.2f | %8.2f | %11.2f | %11.2f | %10.2f | %11.2f |\n",
                l->name,
                l->previous,
                l->current,
                l->consumption,
                l->before_tax,
                l->vat,
                l->total
            );

        }
        else {

            fprintf(
                out,
                "| %-20.20s | %8s | %8s | %11s | %11.2f | %10.2f | %11.2f |\n",
                l->name,
                "-",
                "-",
                "-",
                l->before_tax,
                l->vat,
                l->total
            );

        }

    }

    fprintf(out, "%s", sep);

    fprintf(
        out,
        "\nTOTAL BEFORE TAX       : %12.2f DA\n",
        b->total_before_tax
    );

    fprintf(
        out,
        "TOTAL VAT              : %12.2f DA\n",
        b->total_vat
    );

    fprintf(
        out,
        "SUBTOTAL               : %12.2f DA\n",
        b->total_before_discount
    );

    fprintf(
        out,
        "DISCOUNT (%.2f%%)       : %12.2f DA\n",
        b->discount_rate,
        b->discount_amount
    );

    fprintf(
        out,
        "============================================================\n"
    );

    fprintf(
        out,
        "TOTAL AMOUNT TO PAY    : %12.2f DA\n",
        b->total_to_pay
    );

    fprintf(
        out,
        "============================================================\n"
    );

    fprintf(
        out,
        "\nThank you for choosing FactureUnique!\n"
    );
}


/* =====================================================================
   SECTION 11 - SAVE INVOICE TO FILE
   ===================================================================== */


/* Save invoice using its unique ID as filename */

void save_bill(const Bill *b)
{
    char filename[100];

    FILE *f;

    snprintf(
        filename,
        sizeof(filename),
        "invoice_%s.txt",
        b->invoice_id
    );

    f = fopen(filename, "w");

    if (f == NULL) {

        printf(
            "\n[!] Could not create invoice file.\n"
        );

        return;

    }

    display_bill(b, f);

    if (fclose(f) != 0) {

        printf("\n[!] Error while saving the invoice.\n");

        return;

    }

    printf(
        "\n[+] Invoice successfully saved: %s\n",
        filename
    );
}


/* =====================================================================
   SECTION 12 - INTERACTIVE MENU
   ===================================================================== */


void print_menu(const int added[])
{
    printf("\n");
    printf("================================================\n");
    printf("                 SERVICES MENU                  \n");
    printf("================================================\n");

    printf(
        " 1. Electricity            %s\n",
        added[S_ELEC] ? "[ADDED]" : ""
    );

    printf(
        " 2. Gas                    %s\n",
        added[S_GAS] ? "[ADDED]" : ""
    );

    printf(
        " 3. Water                  %s\n",
        added[S_WATER] ? "[ADDED]" : ""
    );

    printf(
        " 4. Telephone & Internet   %s\n",
        added[S_PHONE] ? "[ADDED]" : ""
    );

    printf(
        " 5. Waste collection       %s\n",
        added[S_OTHER] ? "[ADDED]" : ""
    );

    printf("------------------------------------------------\n");

    printf(" 6. Generate invoice\n");

    printf(" 7. View billing history\n");

    printf(" 0. Cancel current invoice\n");

    printf("================================================\n");
}


/* =====================================================================
   SECTION 13 - MAIN PROGRAM
   ===================================================================== */


int main(void)
{
    int again = 1;

    BillHistory history;

    history_init(&history);

    printf("\n");

    printf("============================================================\n");

    printf("                                                            \n");

    printf("                    FACTUREUNIQUE                           \n");

    printf("             SMART UTILITY BILLING SYSTEM                   \n");

    printf("                                                            \n");

    printf("              ASDS MINI PROJECT - ESTIN                     \n");

    printf("                                                            \n");

    printf("============================================================\n");


    while (again) {

        Bill b;

        int added[6] = {0};

        int choice;

        int done = 0;

        int cancelled = 0;


        bill_init(&b);

        generate_invoice_id(&b);


        /* Customer information */

        printf("\n");

        printf("============== CUSTOMER INFORMATION ==============\n");

        read_string(
            "Customer name   : ",
            b.customer,
            STR_LEN
        );

        read_string(
            "Account number  : ",
            b.account,
            STR_LEN
        );

        read_string(
            "Billing period  : ",
            b.period,
            STR_LEN
        );


        /* Service selection */

        while (!done) {

            print_menu(added);

            choice = read_int(
                "Your choice: ",
                0,
                7
            );


            /* Check for duplicate services */

            if (choice >= S_ELEC &&
                choice <= S_OTHER &&
                added[choice]) {

                printf(
                    "\n[!] This service has already been added.\n"
                );

                continue;

            }


            switch (choice) {

                case S_ELEC:

                    add_electricity(&b);

                    added[choice] = 1;

                    break;


                case S_GAS:

                    add_gas(&b);

                    added[choice] = 1;

                    break;


                case S_WATER:

                    add_water(&b);

                    added[choice] = 1;

                    break;


                case S_PHONE:

                    add_phone(&b);

                    added[choice] = 1;

                    break;


                case S_OTHER:

                    add_other(&b);

                    added[choice] = 1;

                    break;


                case 6:

                    if (b.count == 0) {

                        printf(
                            "\n[!] Add at least one service first.\n"
                        );

                    }
                    else {

                        done = 1;

                    }

                    break;


                case 7:

                    display_history(&history);

                    break;


                case 0:

                    cancelled = 1;

                    done = 1;

                    break;

            }

        }


        /* Invoice generation */

        if (!cancelled) {

            printf("\n");

            printf("============== INVOICE GENERATION ==============\n");


            /* Discount management */

            b.discount_rate = read_percentage(
                "Enter discount percentage (0-100): "
            );


            /* Calculate all totals */

            bill_compute_totals(&b);


            /* Payment status */

            printf("\n");

            b.is_paid = read_yes_no(
                "Has the customer paid this invoice?"
            );


            /* Display invoice */

            display_bill(&b, stdout);


            /* Smart analysis */

            analyze_consumption(&b);


            /* Compare with previous invoice */

            compare_previous_bill(&history, &b);


            /* Save invoice */

            if (read_yes_no(
                "\nSave this invoice in a text file?"
            )) {

                save_bill(&b);

            }


            /* Add invoice to history */

            history_add(&history, &b);


            printf("\n[+] Invoice added to billing history.\n");

        }
        else {

            printf("\n[!] Current invoice cancelled.\n");

        }


        /* Continue or exit */

        again = read_yes_no(
            "\nWould you like to issue another invoice?"
        );

    }


    /* Final summary */

    printf("\n");

    printf("============================================================\n");

    printf("                  FACTUREUNIQUE SUMMARY                     \n");

    printf("============================================================\n");

    printf(
        "Total invoices generated: %d\n",
        history.count
    );

    printf("\nThank you for using FactureUnique!\n");

    printf("Have a great day.\n");

    printf("============================================================\n");


    return 0;
}


/* ======================== END OF PROGRAM ======================== */
