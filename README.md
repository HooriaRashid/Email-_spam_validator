#include <stdio.h>
#include <stdlib.h>
#define TAPE_SIZE 4096

typedef enum {
    q_init,
    q_from,
    q_check_subject_start,
    q_subject,
    q_check_body_start,
    q_body,
    q_evaluate_score,
    q_accept_safe,  
    q_halt_error
} State;

struct TuringMachine {
    char tape[TAPE_SIZE];
    int head;
    State current_state;
    int spam_score;
    int machine_fault;
};

const char* spam_domains[] = {
    "spam.com", "win.net", "lottery.org", "prizemoney.com", "casino-deals.com",
    "jackpotmail.net", "free-cash.cc", "earn-fast.biz", "crypto-moon.info", "cheap-v1agra.ru",
    "mega-bonus.xyz", "claim-reward.top", "giftcard-center.online", "rich-quick.work", "inherited-wealth.co"
};

const char* trusted_domains[] = {
    "hod.edu", "admin.org", "university.edu", "safe.com", "faculty.edu",
    "dean-office.edu", "registrar.edu", "ieee.org", "acm.org", "github.com",
    "scholar.edu", "researchgate.net", "government.gov", "edusecure.org", "alumni.edu"
};

const char* spam_keywords[] = {
    "money", "prize", "urgent", "free", "claim", "m0ney!", "cash", "lottery",
    "jackpot", "crypto", "bitcoin", "investment", "inherited", "millions", "dollars",
    "earning", "wealth", "refund", "act-now", "immediate", "expires", "last-chance",
    "selected", "winner", "congratulations", "giftcard", "reward", "verify-account",
    "password-expired", "unauthorized-login", "action-required", "billing", "viagra",
    "v1agra", "c1al1s", "f-r-e-e", "pr1ze", "cr7pto", "b1tco1n", "cl@im", "m0ney",
    "urg3nt", "priz3", "100%-guaranteed", "no-risk", "risk-free", "hidden-fees",
    "cancel-anytime", "double-income", "get-rich", "make-money", "cheap-deals",
    "lowest-price", "shopper-rewards", "click-here"
};

int spam_domains_count     = sizeof(spam_domains) / sizeof(spam_domains[0]);
int trusted_domains_count   = sizeof(trusted_domains) / sizeof(trusted_domains[0]);
int spam_keywords_count    = sizeof(spam_keywords) / sizeof(spam_keywords[0]);

void print_box_top(const char* label) {
    printf("\n+--------------------------------------------------+\n");
    printf("|  %-48s|\n", label);
    printf("+--------------------------------------------------+\n");
}
void print_box_bottom() {
    printf("+--------------------------------------------------+\n\n");
}
void print_box_line(const char* msg) {
    printf("|  %-48s|\n", msg);
}

int custom_strlen(const char* str) {
    int len = 0;
    while (str[len] != '\0') {
        len++;
    }
    return len;
}

void remove_newline(char* str) {
    int i = 0;
    while (str[i] != '\0') {
        if (str[i] == '\n' || str[i] == '\r') {
            str[i] = '\0';
            break;
        }
        i++;
    }
}

int custom_strstr(const char* tape_segment, const char* keyword) {
    int i = 0, j = 0;
    while (tape_segment[i] != '\0') {
        j = 0;
        while (keyword[j] != '\0' && tape_segment[i + j] == keyword[j]) {
            j++;
        }
        if (keyword[j] == '\0') {
            return 1;
        }
        i++;
    }
    return 0;
}

void load_email(struct TuringMachine* tm, const char* from, const char* subject, const char* body) {
    int i = 0, j = 0;
    
    tm->tape[i++] = 'F'; tm->tape[i++] = 'R'; tm->tape[i++] = 'O';
    tm->tape[i++] = 'M'; tm->tape[i++] = ':';
    j = 0;
    while (from[j] != '\0') tm->tape[i++] = from[j++];
    tm->tape[i++] = '#';
    
    tm->tape[i++] = 'S'; tm->tape[i++] = 'U'; tm->tape[i++] = 'B';
    tm->tape[i++] = 'J'; tm->tape[i++] = 'E'; tm->tape[i++] = 'C';
    tm->tape[i++] = 'T'; tm->tape[i++] = ':';
    j = 0;
    while (subject[j] != '\0') tm->tape[i++] = subject[j++];
    tm->tape[i++] = '#';
    
    tm->tape[i++] = 'B'; tm->tape[i++] = 'O'; tm->tape[i++] = 'D';
    tm->tape[i++] = 'Y'; tm->tape[i++] = ':';
    j = 0;
    while (body[j] != '\0') tm->tape[i++] = body[j++];
    tm->tape[i++] = '#';
    tm->tape[i] = '\0';
    
    tm->head          = 0;
    tm->current_state = q_init;
    tm->spam_score    = 0;
    tm->machine_fault = 0;
    printf("\n[System Log] Tape initialized with user content:\n\"%s\"\n\n", tm->tape);
}

void read_word_at_head(struct TuringMachine* tm, char* buffer) {
    int temp_head = tm->head;
    int b_idx = 0;
    while (tm->tape[temp_head] != '\0' &&
           tm->tape[temp_head] != ' '  &&
           tm->tape[temp_head] != '#'  &&
           tm->tape[temp_head] != ':') {
        buffer[b_idx++] = tm->tape[temp_head];
        temp_head++;
    }
    buffer[b_idx] = '\0';
}

void overwrite_word_with_asterisk(struct TuringMachine* tm, int length) {
    int k;
    for (k = 0; k < length; k++) {
        if (tm->tape[tm->head] != '\0' &&
            tm->tape[tm->head] != '#'  &&
            tm->tape[tm->head] != ' ') {
            char msg[64];
            snprintf(msg, sizeof(msg), "[TM Write] '%c' overwritten with '*'", tm->tape[tm->head]);
            print_box_line(msg);
            tm->tape[tm->head] = '*';
        }
        tm->head++;
    }
}

void skip_whitespace_or_delimiter(struct TuringMachine* tm) {
    while (tm->tape[tm->head] == ' ' || tm->tape[tm->head] == ':') {
        tm->head++;
    }
}

void run_turing_machine(struct TuringMachine* tm) {
    char word_buffer[256];
    char current_char;
    int s, t, w, c;
    int is_spam_dom, is_trusted, is_spam_wd, has_leetspeak;
    char msg[128];
    int box_open = 0; 

    while (tm->current_state != q_accept_safe &&
           tm->current_state != q_halt_error) {
           
        if (tm->head >= TAPE_SIZE || tm->tape[tm->head] == '\0') {
            if (tm->current_state == q_body || tm->current_state == q_evaluate_score) {
                tm->current_state = q_evaluate_score;
            } else {
                tm->current_state = q_halt_error;
                tm->machine_fault = 1;
                if (box_open) { print_box_bottom(); box_open = 0; }
                printf("[Error] Sudden head crash. Out of tape boundary.\n");
                break;
            }
        }
        
        current_char = tm->tape[tm->head];
        
        if (tm->current_state == q_init) {
            read_word_at_head(tm, word_buffer);
            if (custom_strstr(word_buffer, "FROM")) {
                tm->head += 4;
                skip_whitespace_or_delimiter(tm);
                tm->current_state = q_from;
                print_box_top("[ FROM FIELD ANALYSIS ]");
                box_open = 1;
                print_box_line("[State Shift] Transitioned to q_from.");
            } else {
                tm->head++;
            }
        }
        
        else if (tm->current_state == q_from) {
            if (current_char == '#') {
                tm->head++;
                tm->current_state = q_check_subject_start;
                print_box_line("[State Shift] FROM done. Moving to Subject.");
                print_box_bottom();
                box_open = 0;
                continue;
            }
            if (current_char == '@') {
                tm->head++;
                read_word_at_head(tm, word_buffer);
                snprintf(msg, sizeof(msg), "[Analysis] Domain: %s", word_buffer);
                print_box_line(msg);
                is_spam_dom = 0;
                for (s = 0; s < spam_domains_count; s++) {
                    if (custom_strstr(word_buffer, spam_domains[s])) {
                        is_spam_dom = 1;
                        break;
                    }
                }
                if (is_spam_dom) {
                    tm->spam_score += 15;
                    snprintf(msg, sizeof(msg), "[BLACKLISTED] Score +15 => Total: %d", tm->spam_score);
                    print_box_line(msg);
                    overwrite_word_with_asterisk(tm, custom_strlen(word_buffer));
                } else {
                    is_trusted = 0;
                    for (t = 0; t < trusted_domains_count; t++) {
                        if (custom_strstr(word_buffer, trusted_domains[t])) {
                            is_trusted = 1;
                            break;
                        }
                    }
                    if (is_trusted) {
                        print_box_line("[TRUSTED] Domain verified as safe.");
                    }
                    tm->head += custom_strlen(word_buffer);
                }
            } else {
                tm->head++;
            }
        }
        
        else if (tm->current_state == q_check_subject_start) {
            read_word_at_head(tm, word_buffer);
            if (custom_strstr(word_buffer, "SUBJECT")) {
                tm->head += 7; 
                skip_whitespace_or_delimiter(tm);
                tm->current_state = q_subject;
                print_box_top("[ SUBJECT FIELD ANALYSIS ]");
                box_open = 2;
                print_box_line("[State Shift] Transitioned to q_subject.");
            } else {
                tm->head++;
            }
        }
        
        else if (tm->current_state == q_subject) {
            if (current_char == '#') {
                tm->head++;
                tm->current_state = q_check_body_start;
                print_box_line("[State Shift] SUBJECT done. Moving to Body.");
                print_box_bottom();
                box_open = 0;
                continue;
            }
            skip_whitespace_or_delimiter(tm);
            read_word_at_head(tm, word_buffer);
            if (custom_strlen(word_buffer) == 0) {
                tm->head++;
                continue;
            }
            has_leetspeak = 0;
            for (c = 0; word_buffer[c] != '\0'; c++) {
                char ch = word_buffer[c];
                if (ch == '0' || ch == '1' || ch == '3' || ch == '!' || ch == '$') {
                    has_leetspeak = 1;
                    break;
                }
            }
            is_spam_wd = 0;
            for (w = 0; w < spam_keywords_count; w++) {
                if (custom_strstr(word_buffer, spam_keywords[w])) {
                    is_spam_wd = 1;
                    break;
                }
            }
            if (is_spam_wd || has_leetspeak) {
                if (has_leetspeak) {
                    tm->spam_score += 5;
                    print_box_line("[OBFUSCATION DETECTED] Score +5.");
                }
                tm->spam_score += 15;
                snprintf(msg, sizeof(msg), "[SPAM WORD] Flagged. Score => %d", tm->spam_score);
                print_box_line(msg);
                overwrite_word_with_asterisk(tm, custom_strlen(word_buffer));
            } else {
                snprintf(msg, sizeof(msg), "[Safe] Word '%s' kept intact.", word_buffer);
                print_box_line(msg);
                tm->head += custom_strlen(word_buffer);
            }
        }
        
        else if (tm->current_state == q_check_body_start) {
            read_word_at_head(tm, word_buffer);
            if (custom_strstr(word_buffer, "BODY")) {
                tm->head += 4;
                skip_whitespace_or_delimiter(tm);
                tm->current_state = q_body;
                print_box_top("[ BODY FIELD ANALYSIS ]");
                box_open = 3;
                print_box_line("[State Shift] Transitioned to q_body.");
            } else {
                tm->head++;
            }
        }
        
        else if (tm->current_state == q_body) {
            if (current_char == '#' || current_char == '\0') {
                tm->current_state = q_evaluate_score;
                print_box_line("[State Shift] Body scan done. Evaluating score.");
                print_box_bottom();
                box_open = 0;
                continue;
            }
            skip_whitespace_or_delimiter(tm);
            read_word_at_head(tm, word_buffer);
            if (custom_strlen(word_buffer) == 0) {
                tm->head++;
                continue;
            }
            has_leetspeak = 0;
            for (c = 0; word_buffer[c] != '\0'; c++) {
                char ch = word_buffer[c];
                if (ch == '0' || ch == '1' || ch == '3' || ch == '!' || ch == '$') {
                    has_leetspeak = 1;
                    break;
                }
            }
            is_spam_wd = 0;
            for (w = 0; w < spam_keywords_count; w++) {
                if (custom_strstr(word_buffer, spam_keywords[w])) {
                    is_spam_wd = 1;
                    break;
                }
            }
            if (is_spam_wd || has_leetspeak) {
                if (has_leetspeak) {
                    tm->spam_score += 5;
                    print_box_line("[OBFUSCATION DETECTED] In Body. Score +5.");
                }
                tm->spam_score += 15;
                snprintf(msg, sizeof(msg), "[SPAM WORD] Flagged in Body. Score => %d", tm->spam_score);
                print_box_line(msg);
                overwrite_word_with_asterisk(tm, custom_strlen(word_buffer));
            } else {
                snprintf(msg, sizeof(msg), "[Safe] Word '%s' kept intact.", word_buffer);
                print_box_line(msg);
                tm->head += custom_strlen(word_buffer);
            }
        }
        
        else if (tm->current_state == q_evaluate_score) {
            printf("[Metric Check] Cumulative Score: %d\n", tm->spam_score);
            
            if (tm->spam_score >= 40) {
                tm->current_state = q_accept_safe; 
                printf("\n>>> Score >= 40. EMAIL IS SPAM! <<<\n");
            } else {
                printf("\n>>>  Score < 40. Machine halt EMAIL IS SAFE. <<<\n");
                break; 
            }
        }
    }
}

int main() {
    struct TuringMachine detector;
    char input_from[256];
    char input_subject[256];
    char input_body[1024];
    
    printf("===================================================\n");
    printf("     SIMULATION: TURING MACHINE SPAM DETECTOR      \n");
    printf("===================================================\n\n");
    
    printf("Enter Sender Email:  ");
    fgets(input_from, sizeof(input_from), stdin);
    remove_newline(input_from);
    
    printf("Enter Email Subject:  ");
    fgets(input_subject, sizeof(input_subject), stdin);
    remove_newline(input_subject);
    
    printf("Enter Email Body Context: ");
    fgets(input_body, sizeof(input_body), stdin);
    remove_newline(input_body);
    
    printf("\n>>> INITIALIZING TURING TAPE STREAM <<<");
    load_email(&detector, input_from, input_subject, input_body);
    
    printf(">>> RUNNING MACHINE TRANSITIONS <<<\n");
    run_turing_machine(&detector);
    
    printf("\n===================================================\n");
    printf("                  FINAL EXECUTION REPORT            \n");
    printf("===================================================\n");
    
    if (detector.current_state == q_accept_safe) {
        printf("Target Halt State  : q_accept_safe (HALTED - EMAIL IS SPAM)\n");
    } else if (detector.current_state == q_evaluate_score) {
        printf("Target Halt State  : q_evaluate_score (HALTED - EMAIL IS SAFE)\n");
    } else {
        printf("Target Halt State     : q_halt_error\n");
    }
    
    printf("Final Machine Score   : %d\n", detector.spam_score);
    printf("Modified Tape :\n\"%s\"\n", detector.tape);
    printf("===================================================\n");
    
    return 0;
}
