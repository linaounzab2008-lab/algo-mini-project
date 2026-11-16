
#ifndef BILL_H
#define BILL_H

#define MAX_BILL_LINES 4

typedef struct {
    char service[32];
    char details[128];
    double amount_before_tax;
    double vat_rate;
} BillLine;

/* Service calculations */
double electricity_amount(double consumption);
double gas_amount(double consumption);
double water_amount(double consumption);
double phone_amount(double minutes);
double internet_amount(int package_mbps);

/* VAT calculation */
double vat_amount(double amount_before_tax, double vat_rate);

/* Display and save invoice */
void display_bill(
    const char *name,
    const char *account_number,
    const char *billing_period,
    const BillLine lines[],
    int line_count
);

int save_bill(
    const char *name,
    const char *account_number,
    const char *billing_period,
    const BillLine lines[],
    int line_count
);

#endif

#include "bill.h"

#include <stdio.h>
#include <time.h>
#include <ctype.h>

/* Original fixed fees */
#define ELECTRICITY_FIXED_FEE 100.0
#define GAS_FIXED_FEE          50.0
#define WATER_FIXED_FEE       100.0

/* Original VAT rates */
#define ELECTRICITY_VAT_RATE 0.09
#define GAS_VAT_RATE         0.09
#define WATER_VAT_RATE       0.09
#define PHONE_VAT_RATE       0.19
#define INTERNET_VAT_RATE    0.19

/* Telephone tariff */
#define PHONE_SUBSCRIPTION      600.0
#define PHONE_INCLUDED_MINUTES  100.0
#define PHONE_EXTRA_RATE        3.0

/* Water sanitation */
#define WATER_SANITATION_RATE 0.10


/* Progressive tariff calculation */
static double progressive_amount(
    double consumption,
    double first_limit,
    double second_limit,
    double rate1,
    double rate2,
    double rate3
)
{
    double amount = 0.0;

    if (consumption <= first_limit) {
        amount = consumption * rate1;
    }
    else if (consumption <= second_limit) {
        amount = first_limit * rate1
               + (consumption - first_limit) * rate2;
    }
    else {
        amount = first_limit * rate1
               + (second_limit - first_limit) * rate2
               + (consumption - second_limit) * rate3;
    }

    return amount;
}


/* Electricity */
double electricity_amount(double consumption)
{
    return progressive_amount(
        consumption, 100.0, 300.0, 2.0, 4.0, 6.0
    ) + ELECTRICITY_FIXED_FEE;
}


/* Gas */
double gas_amount(double consumption)
{
    return progressive_amount(
        consumption, 100.0, 300.0, 1.5, 3.0, 4.5
    ) + GAS_FIXED_FEE;
}


/* Water */
double water_amount(double consumption)
{
    double water_usage;
    double sanitation_fee;

    water_usage = progressive_amount(
        consumption, 20.0, 40.0, 6.0, 20.0, 40.0
    );

    sanitation_fee = water_usage * WATER_SANITATION_RATE;

    return water_usage + sanitation_fee + WATER_FIXED_FEE;
}


/* Telephone */
double phone_amount(double minutes)
{
    double extra_minutes = minutes - PHONE_INCLUDED_MINUTES;

    if (extra_minutes < 0.0) {
        extra_minutes = 0.0;
    }

    return PHONE_SUBSCRIPTION + extra_minutes * PHONE_EXTRA_RATE;
}


/* Internet */
double internet_amount(int package_mbps)
{
    switch (package_mbps) {
        case 4:
            return 1000.0;

        case 8:
            return 1500.0;

        case 20:
            return 2500.0;

        default:
            return 0.0;
    }
}


/* VAT */
double vat_amount(double amount_before_tax, double vat_rate)
{
    return amount_before_tax * vat_rate;
}


/*
   Common invoice function.
   It can print the invoice on the screen
   or write it into a text file.
*/
static void print_bill(
    FILE *out,
    const char *name,
    const char *account_number,
    const char *billing_period,
    const BillLine lines[],
    int line_count
)
{
    double total_before_tax = 0.0;
    double total_vat = 0.0;
    double vat;
    double total;
    int i;

    fprintf(out, "\n");
    fprintf(out, "======================================================================================\n");
    fprintf(out, "                    UNIFIED UTILITY BILL\n");
    fprintf(out, "======================================================================================\n");

    fprintf(out, "Customer : %s\n", name);
    fprintf(out, "Account  : %s\n", account_number);
    fprintf(out, "Period   : %s\n", billing_period);

    fprintf(out, "--------------------------------------------------------------------------------------\n");

    fprintf(out,
        "| %-14s | %-12s | %10s | %8s | %10s |\n",
        "SERVICE", "DETAILS", "BEFORE TAX", "VAT", "TOTAL"
    );

    fprintf(out, "--------------------------------------------------------------------------------------\n");

    for (i = 0; i < line_count; i++) {

        vat = vat_amount(
            lines[i].amount_before_tax,
            lines[i].vat_rate
        );

        total = lines[i].amount_before_tax + vat;

        fprintf(out,
            "| %-14.14s | %-12.12s | %10.2f | %8.2f | %10.2f |\n",
            lines[i].service,
            lines[i].details,
            lines[i].amount_before_tax,
            vat,
            total
        );

        total_before_tax += lines[i].amount_before_tax;
        total_vat += vat;
    }

    fprintf(out, "--------------------------------------------------------------------------------------\n");

    fprintf(out,
        "| %-14s | %-12s | %10.2f | %8.2f | %10.2f |\n",
        "TOTAL", "",
        total_before_tax,
        total_vat,
        total_before_tax + total_vat
    );

    fprintf(out, "======================================================================================\n");

    fprintf(out, "\n");
    fprintf(out, "                  PAYMENT SUMMARY\n");
    fprintf(out, "--------------------------------------------------------------------------------------\n");

    fprintf(out, "Subtotal       : %.2f DA\n", total_before_tax);
    fprintf(out, "Total VAT      : %.2f DA\n", total_vat);

    fprintf(out, "--------------------------------------------------------------------------------------\n");

    fprintf(out, "TOTAL TO PAY   : %.2f DA\n",
            total_before_tax + total_vat);

    fprintf(out, "======================================================================================\n");
    fprintf(out, "             Thank you for using our service!\n");
    fprintf(out, "======================================================================================\n");
}


/* Display invoice */
void display_bill(
    const char *name,
    const char *account_number,
    const char *billing_period,
    const BillLine lines[],
    int line_count
)
{
    print_bill(
        stdout, name, account_number,
        billing_period, lines, line_count
    );
}


/* Save invoice */
int save_bill(
    const char *name,
    const char *account_number,
    const char *billing_period,
    const BillLine lines[],
    int line_count
)
{
    char filename[160];
    char safe_account[80];
    int i;

    FILE *file;
    time_t now;
    struct tm *date;

    /* Make account number safe for filenames */
    for (i = 0; account_number[i] != '\0' && i < 79; i++) {

        if (isalnum((unsigned char)account_number[i])) {
            safe_account[i] = account_number[i];
        }
        else {
            safe_account[i] = '_';
        }
    }

    safe_account[i] = '\0';

    /* Generate date and time */
    now = time(NULL);
    date = localtime(&now);

    if (date == NULL) {
        printf("Could not get the current date.\n");
        return 0;
    }

    snprintf(
        filename,
        sizeof(filename),
        "invoice_%s_%04d%02d%02d_%02d%02d%02d.txt",
        safe_account,
        date->tm_year + 1900,
        date->tm_mon + 1,
        date->tm_mday,
        date->tm_hour,
        date->tm_min,
        date->tm_sec
    );

    file = fopen(filename, "w");

    if (file == NULL) {
        printf("Error: Unable to save the invoice.\n");
        return 0;
    }

    /* Write the complete invoice */
    print_bill(
        file, name, account_number,
        billing_period, lines, line_count
    );

    fclose(file);

    printf("\nInvoice saved successfully!\n");
    printf("File name: %s\n", filename);

    return 1;
}

#include "bill.h"

#include <ctype.h>
#include <errno.h>
#include <math.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define INPUT_SIZE 256

#define ELECTRICITY_VAT_RATE 0.09
#define GAS_VAT_RATE         0.09
#define WATER_VAT_RATE       0.09
#define PHONE_VAT_RATE       0.19


/* Remove newline */
static void remove_newline(char *text)
{
    size_t length = strlen(text);

    if (length > 0 && text[length - 1] == '\n') {
        text[length - 1] = '\0';
    }
}


/* Read text */
static void read_text(
    const char *prompt,
    char *output,
    size_t output_size
)
{
    while (1) {

        printf("%s", prompt);

        if (fgets(output, (int)output_size, stdin) == NULL) {
            printf("\nInput ended. Exiting.\n");
            exit(EXIT_FAILURE);
        }

        remove_newline(output);

        if (output[0] != '\0') {
            return;
        }

        printf("Input cannot be empty. Please try again.\n");
    }
}


/* Read a non-negative number */
static double read_nonnegative_double(const char *prompt)
{
    char input[INPUT_SIZE];
    char *end;
    double value;

    while (1) {

        printf("%s", prompt);

        if (fgets(input, sizeof(input), stdin) == NULL) {
            printf("\nInput ended. Exiting.\n");
            exit(EXIT_FAILURE);
        }

        errno = 0;
        value = strtod(input, &end);

        while (isspace((unsigned char)*end)) {
            end++;
        }

        if (end == input || *end != '\0' ||
            errno == ERANGE || !isfinite(value) ||
            value < 0.0) {

            printf("Please enter a valid non-negative number.\n");
            continue;
        }

        return value;
    }
}


/* Read an integer */
static int read_integer(
    const char *prompt,
    int minimum,
    int maximum
)
{
    char input[INPUT_SIZE];
    char *end;
    long value;

    while (1) {

        printf("%s", prompt);

        if (fgets(input, sizeof(input), stdin) == NULL) {
            printf("\nInput ended. Exiting.\n");
            exit(EXIT_FAILURE);
        }

        errno = 0;
        value = strtol(input, &end, 10);

        while (isspace((unsigned char)*end)) {
            end++;
        }

        if (end == input || *end != '\0' ||
            errno == ERANGE ||
            value < minimum || value > maximum) {

            printf("Please enter a whole number from %d to %d.\n",
                   minimum, maximum);
            continue;
        }

        return (int)value;
    }
}


/* Read previous and current meter readings */
static void read_indices(double *previous, double *current)
{
    while (1) {

        *previous = read_nonnegative_double("Previous index: ");
        *current = read_nonnegative_double("Current index: ");

        if (*current >= *previous) {
            return;
        }

        printf("The current index cannot be lower than the previous index.\n");
        printf("Please enter both readings again.\n");
    }
}


/* Add a service to the bill */
static void add_line(
    BillLine lines[],
    int *line_count,
    const char *service,
    const char *details,
    double amount,
    double vat_rate
)
{
    BillLine *line = &lines[*line_count];

    snprintf(line->service, sizeof(line->service), "%s", service);
    snprintf(line->details, sizeof(line->details), "%s", details);

    line->amount_before_tax = amount;
    line->vat_rate = vat_rate;

    (*line_count)++;
}


/* Electricity */
static void add_electricity(
    BillLine lines[],
    int *line_count
)
{
    double previous;
    double current;
    double consumption;
    char details[128];

    printf("\nElectricity meter readings (kWh)\n");

    read_indices(&previous, &current);

    consumption = current - previous;

    snprintf(details, sizeof(details),
             "Used: %.2f kWh", consumption);

    add_line(
        lines, line_count,
        "Electricity",
        details,
        electricity_amount(consumption),
        ELECTRICITY_VAT_RATE
    );
}


/* Gas */
static void add_gas(
    BillLine lines[],
    int *line_count
)
{
    double previous;
    double current;
    double consumption;
    char details[128];

    printf("\nGas meter readings (m3)\n");

    read_indices(&previous, &current);

    consumption = current - previous;

    snprintf(details, sizeof(details),
             "Used: %.2f m3", consumption);

    add_line(
        lines, line_count,
        "Gas",
        details,
        gas_amount(consumption),
        GAS_VAT_RATE
    );
}


/* Water */
static void add_water(
    BillLine lines[],
    int *line_count
)
{
    double previous;
    double current;
    double consumption;
    char details[128];

    printf("\nWater meter readings (m3)\n");

    read_indices(&previous, &current);

    consumption = current - previous;

    snprintf(details, sizeof(details),
             "Used: %.2f m3", consumption);

    add_line(
        lines, line_count,
        "Water",
        details,
        water_amount(consumption),
        WATER_VAT_RATE
    );
}


/* Telephone and Internet */
static void add_phone_and_internet(
    BillLine lines[],
    int *line_count
)
{
    double previous;
    double current;
    double minutes;
    int package;
    int speed;
    double amount;
    char details[128];

    printf("\nTelephone and Internet\n");

    printf("Telephone meter readings (minutes)\n");

    read_indices(&previous, &current);

    minutes = current - previous;

    printf("\nInternet packages:\n");
    printf("  1. 4 Mbps  - 1000 DA\n");
    printf("  2. 8 Mbps  - 1500 DA\n");
    printf("  3. 20 Mbps - 2500 DA\n");

    package = read_integer("Choose a package (1-3): ", 1, 3);

    if (package == 1) {
        speed = 4;
    }
    else if (package == 2) {
        speed = 8;
    }
    else {
        speed = 20;
    }

    amount = phone_amount(minutes);
    amount += internet_amount(speed);

    snprintf(
        details,
        sizeof(details),
        "%.2f min, %d Mbps",
        minutes,
        speed
    );

    add_line(
        lines, line_count,
        "Phone + Internet",
        details,
        amount,
        PHONE_VAT_RATE
    );
}


/* Ask whether to save the invoice */
static void ask_to_save_bill(
    const char *name,
    const char *account_number,
    const char *billing_period,
    const BillLine lines[],
    int line_count
)
{
    int choice;

    printf("\n");
    printf("======================================================================================\n");
    printf("           SAVE YOUR INVOICE\n");
    printf("======================================================================================\n");
    printf("1. Yes, save invoice\n");
    printf("0. No, continue without saving\n");
    printf("======================================================================================\n");

    choice = read_integer("Your choice: ", 0, 1);

    if (choice == 1) {

        save_bill(
            name,
            account_number,
            billing_period,
            lines,
            line_count
        );
    }
    else {
        printf("\nInvoice was not saved.\n");
    }
}


/* Ask whether to create another bill */
static int ask_to_start_another_bill(void)
{
    printf("\n");

    return read_integer(
        "Create another bill? (1 = Yes, 0 = No): ",
        0, 1
    );
}


/* MAIN PROGRAM */
int main(void)
{
    while (1) {

        char name[INPUT_SIZE];
        char account_number[INPUT_SIZE];
        char billing_period[INPUT_SIZE];

        BillLine lines[MAX_BILL_LINES];

        int line_count = 0;
        int selected[5] = {0};
        int choice;

        printf("\n");
        printf("======================================================================================\n");
        printf("       UNIFIED UTILITY BILL SYSTEM\n");
        printf("======================================================================================\n");

        /* Customer information */
        read_text(
            "Customer name: ",
            name,
            sizeof(name)
        );

        read_text(
            "Account number: ",
            account_number,
            sizeof(account_number)
        );

        read_text(
            "Billing period (e.g. October 2026): ",
            billing_period,
            sizeof(billing_period)
        );

        /* Service selection */
        while (1) {

            printf("\n");
            printf("======================================================================================\n");
            printf("           SERVICE MENU\n");
            printf("======================================================================================\n");

            printf("1. Electricity\n");
            printf("2. Gas\n");
            printf("3. Water\n");
            printf("4. Telephone and Internet\n");
            printf("5. Finish and display bill\n");

            printf("======================================================================================\n");

            choice = read_integer(
                "Your choice (1-5): ",
                1, 5
            );

            if (choice == 5) {

                if (line_count == 0) {
                    printf("Select at least one service first.\n");
                    continue;
                }

                break;
            }

            if (selected[choice]) {

                printf("That service has already been added.\n");
                continue;
            }

            selected[choice] = 1;

            switch (choice) {

                case 1:
                    add_electricity(lines, &line_count);
                    break;

                case 2:
                    add_gas(lines, &line_count);
                    break;

                case 3:
                    add_water(lines, &line_count);
                    break;

                case 4:
                    add_phone_and_internet(lines, &line_count);
                    break;
            }

            printf("\nService added successfully!\n");
        }

        /* Display invoice */
        display_bill(
            name,
            account_number,
            billing_period,
            lines,
            line_count
        );

        /* Save invoice */
        ask_to_save_bill(
            name,
            account_number,
            billing_period,
            lines,
            line_count
        );

        /* Another bill */
        if (!ask_to_start_another_bill()) {

            printf("\n");
            printf("Thank you for using our billing system!\n");

            break;
        }
    }

    return 0;
}
