package compilerCCP;

import java.util.*;

public class HindiLangCompiler {

enum TokenType{
KEYWORD,IDENTIFIER,NUMBER,FLOAT,STRING,
OPERATOR,RELATIONAL,ASSIGN,
LPAREN,RPAREN,LBRACE,RBRACE,LBRACKET,RBRACKET,
SEMICOLON,COMMA,COLON,DOT,ARROW,EOF,ERROR
}


static class Token{

TokenType type;
String value;
int line;

Token(TokenType type,String value,int line){
this.type=type;
this.value=value;
this.line=line;
}

public String toString(){
return type+" : "+value+" (line "+line+")";
}

}


enum State{
START,IDENTIFIER,NUMBER,STRING,
OPERATOR,COMMENT_SINGLE,COMMENT_MULTI,DONE
}



static class Lexer{


private final String input;

private final List<Token> tokens=new ArrayList<>();

private int pos=0;
private int line=1;


// FIX 1: Multi-word keywords must be matched greedily BEFORE single-word ones.
// Store them sorted longest-first so the greedy match works correctly.
private static final List<String> keywordList = new ArrayList<>(Arrays.asList(

// ========== CUSTOM KEYWORDS (Original) ==========
"शुरू","समाप्त",
"मान",
"पूर्णांक","दशमलव","पाठ","बूलियन",
"यदि","तब","अन्यथा","समाप्तयदि",
"जबतक","समाप्तजबतक",
"दोहराओ","बार","समाप्तदोहराओ",
"दिखाओ","लो",
"कार्य","समाप्तकार्य",
"वापस",
"वर्ग","समाप्तवर्ग",
"निर्माता","समाप्तनिर्माता",
"नया","यह",
"सार्वजनिक","निजी","संरक्षित",
"अमूर्तवर्ग","समाप्तअमूर्तवर्ग",
"अमूर्तकार्य",
"सत्य","असत्य",

// ========== JAVA KEYWORDS IN HINDI ==========
"अगर","तो","नहीं तो","अगर नहीं",
"के लिए","समाप्तके लिए",
"जब तक","समाप्त जब तक",
"करो","समाप्त करो",
"स्विच","समाप्त स्विच",
"मामला","डिफ़ॉल्ट",
"तोड़ो","जारी रखो",
"वापसी",

"रिक्त","पूर्ण संख्या","दशमलव संख्या","वर्ण","सूची",
"लंबी","छोटी","बहुत छोटी","असली",
"वस्तु",

"स्थिर","अंतिम","अमूर्त",
"सिंक्रोनाइज़्ड","अस्थिर","क्षणिक","मूल",
"कड़ा",

"अंतर्फलक","समाप्त अंतर्फलक",
"विस्तार करें","कार्यान्वयन करें",
"सुपर",
"यह क्षेत्र",

"आयात","पैकेज",

"कोशिश","पकड़ो","अंत में",
"फेंको","फेंकता है",
"अपवाद",

"का प्रकार है","उदाहरण है",

"मॉड्यूल","आवश्यकता","निर्यात",
"खुला मॉड्यूल",

"अनुमति","संकीर्ण","संख्यात्मक",
"रिकॉर्ड","समाप्त रिकॉर्ड",

"लैम्ब्डा","धारा",
"मानचित्र","फिल्टर","कम करें",

"var","सर्वनाम",
"अगर-अभिव्यक्ति","स्विच-अभिव्यक्ति"
));

// Sort longest first so multi-word keywords match before their sub-words
private static final List<String> KEYWORDS_SORTED;
private static final Set<String> KEYWORD_SET;

static {
    KEYWORDS_SORTED = new ArrayList<>(keywordList);
    KEYWORDS_SORTED.sort((a, b) -> b.length() - a.length());
    KEYWORD_SET = new HashSet<>(keywordList);
}


Lexer(String input){
this.input=input;
}



List<Token> tokenize(){

State state=State.START;
StringBuilder sb=new StringBuilder();

while(pos<input.length()){

char ch=input.charAt(pos);

switch(state){

case START:

sb.setLength(0);

if(Character.isWhitespace(ch)){
if(ch=='\n') line++;
pos++;
continue;
}

if(ch=='/'&&peek('/')){
state=State.COMMENT_SINGLE;
pos+=2;
continue;
}

if(ch=='/'&&peek('*')){
state=State.COMMENT_MULTI;
pos+=2;
continue;
}

if(ch=='#'){
state=State.COMMENT_SINGLE;
pos++;
continue;
}

if(Character.isDigit(ch))
state=State.NUMBER;
else if(ch=='"'){
state=State.STRING;
pos++;
}
// FIX 2: Try greedy multi-word keyword match FIRST before falling into IDENTIFIER mode
else if(isIdentifierStart(ch)){
// Try to match a keyword (possibly multi-word) starting at pos
String matched = tryMatchKeyword();
if(matched != null){
tokens.add(new Token(TokenType.KEYWORD, matched, line));
pos += matched.length();
} else {
state=State.IDENTIFIER;
}
}
else
state=State.OPERATOR;

break;


case IDENTIFIER:

if(isIdentifierChar(ch)){
sb.append(ch);
pos++;
}
else{
// Identifiers are never keywords here (keywords already matched above)
tokens.add(new Token(TokenType.IDENTIFIER, sb.toString(), line));
state=State.START;
}

break;


case NUMBER:

if(Character.isDigit(ch)||ch=='.'){
sb.append(ch);
pos++;
}
else{
tokens.add(
new Token(
sb.toString().contains(".")
?TokenType.FLOAT:
TokenType.NUMBER,
sb.toString(),
line));
state=State.START;
}

break;


case STRING:

if(ch=='"'){
tokens.add(
new Token(TokenType.STRING,sb.toString(),line));
pos++;
state=State.START;
}
else{
sb.append(ch);
pos++;
}

break;


case COMMENT_SINGLE:

if(ch=='\n') state=State.START;
if(ch=='\n') line++;
pos++;

break;


case COMMENT_MULTI:

if(ch=='*'&&peek('/')){
pos+=2;
state=State.START;
}
else{
if(ch=='\n') line++;
pos++;
}

break;


case OPERATOR:

handleOperator();
state=State.START;

break;

case DONE:
break;

}

}

tokens.add(
new Token(TokenType.EOF,"EOF",line));

return tokens;

}


// FIX 3: Greedy keyword matcher. Tries longest keywords first.
// A keyword match is only valid if the character after it is NOT an identifier char
// (to avoid matching "मान" inside "मानचित्र").
private String tryMatchKeyword(){
for(String kw : KEYWORDS_SORTED){
if(input.startsWith(kw, pos)){
int end = pos + kw.length();
// Make sure the keyword is not followed by an identifier character
if(end >= input.length() || !isIdentifierChar(input.charAt(end))){
return kw;
}
}
}
return null;
}


private boolean isIdentifierStart(char c){
return Character.isLetter(c) || c == '_';
}

private boolean isIdentifierChar(char c){
return Character.isLetterOrDigit(c)
||Character.getType(c)==Character.NON_SPACING_MARK
||Character.getType(c)==Character.COMBINING_SPACING_MARK
||c=='_'||c=='।';
// FIX 4: Removed '-' from identifier chars (it's an operator)
// FIX 5: Removed space — identifiers must NOT contain spaces;
//         multi-word keywords are handled by tryMatchKeyword()
}


private void handleOperator(){

char ch=input.charAt(pos);

if(ch=='='&&peek('=')){
tokens.add(new Token(TokenType.RELATIONAL,"==",line));
pos+=2;
}
else if(ch=='!'&&peek('=')){
tokens.add(new Token(TokenType.RELATIONAL,"!=",line));
pos+=2;
}
else if(ch=='>'&&peek('=')){
tokens.add(new Token(TokenType.RELATIONAL,">=",line));
pos+=2;
}
else if(ch=='<'&&peek('=')){
tokens.add(new Token(TokenType.RELATIONAL,"<=",line));
pos+=2;
}
else if(ch=='='&&!peek('>')){
tokens.add(new Token(TokenType.ASSIGN,"=",line));
pos++;
}
else if(ch=='='&&peek('>')){
tokens.add(new Token(TokenType.ARROW,"=>",line));
pos+=2;
}
// FIX 16: Explicitly identify < and > as RELATIONAL tokens instead of plain OPERATORs
else if(ch=='<' || ch=='>'){
tokens.add(new Token(TokenType.RELATIONAL,""+ch,line));
pos++;
}
// Removed < and > from the catch-all math operator string below
else if("+-*/%".contains(""+ch)){
tokens.add(new Token(TokenType.OPERATOR,""+ch,line));
pos++;
}
else if(ch=='('){
tokens.add(new Token(TokenType.LPAREN,"(",line));
pos++;
}
else if(ch==')'){
tokens.add(new Token(TokenType.RPAREN,")",line));
pos++;
}
else if(ch=='{'){
tokens.add(new Token(TokenType.LBRACE,"{",line));
pos++;
}
else if(ch=='}'){
tokens.add(new Token(TokenType.RBRACE,"}",line));
pos++;
}
else if(ch=='['){
tokens.add(new Token(TokenType.LBRACKET,"[",line));
pos++;
}
else if(ch==']'){
tokens.add(new Token(TokenType.RBRACKET,"]",line));
pos++;
}
else if(ch==';'){
tokens.add(new Token(TokenType.SEMICOLON,";",line));
pos++;
}
else if(ch==','){
tokens.add(new Token(TokenType.COMMA,",",line));
pos++;
}
else if(ch==':'){
tokens.add(new Token(TokenType.COLON,":",line));
pos++;
}
else if(ch=='.'){
tokens.add(new Token(TokenType.DOT,".",line));
pos++;
}
else{
tokens.add(new Token(TokenType.ERROR,""+ch,line));
pos++;
}

}


private boolean peek(char c){
return pos+1<input.length()
&&input.charAt(pos+1)==c;
}


}


// ================ PARSER (Enhanced) =================

static class Parser {

private final List<Token> tokens;
private int current=0;


Parser(List<Token> tokens){
this.tokens=tokens;
}



void parseProgram(){

match("शुरू");

while(!check("समाप्त")&&!isEOF()){

try{
parseStatement();
}
catch(Exception e){
System.out.println(
"Parser Error line "+peek().line+
" : "+e.getMessage());
synchronize();
}

}

match("समाप्त");

System.out.println(
"\n✓ Syntax Analysis Completed");

}


private void parseStatement(){

if(check("मान")||
check("पूर्णांक")||
check("दशमलव")||
check("पाठ")||
check("बूलियन")||
check("पूर्ण संख्या")||
check("दशमलव संख्या")||
check("वर्ण")||
check("लंबी")||
check("छोटी")||
check("बहुत छोटी")||
check("असली")||
check("वस्तु")||
check("var")||
check("सर्वनाम"))
parseDeclaration();

else if(check("दिखाओ"))
parseShow();

else if(check("लो"))
parseInput();

else if(check("यदि")||check("अगर")||check("अगर-अभिव्यक्ति"))
parseIf();

else if(check("जबतक")||check("जब तक"))
parseWhile();

else if(check("दोहराओ"))
parseRepeat();

else if(check("के लिए"))
parseForLoop();

else if(check("करो"))
parseDoWhile();

else if(check("स्विच")||check("स्विच-अभिव्यक्ति"))
parseSwitch();

else if(check("कार्य"))
parseFunction();

else if(check("वर्ग"))
parseClass();

else if(check("अंतर्फलक"))
parseInterface();

else if(check("अमूर्तवर्ग"))
parseAbstractClass();

else if(check("रिकॉर्ड"))
parseRecord();

else if(check("अमूर्तकार्य"))
parseAbstractFunction();

else if(check("वापस")||check("वापसी"))
parseReturn();

else if(check("कोशिश"))
parseTryBlock();

else if(check("फेंको"))
parseThrow();

else if(check("तोड़ो"))
advance();

else if(check("जारी रखो"))
advance();

else if(peek().type==TokenType.IDENTIFIER)
// FIX 6: parseAssignmentOrCall to handle both assignment and function calls
parseAssignmentOrCall();

else
{advance();}

}


private void parseDeclaration(){
advance();
consume(TokenType.IDENTIFIER,"Identifier expected");
match("=");
parseExpression();
}


// FIX 7: Renamed to handle function calls (identifier followed by '()')
private void parseAssignmentOrCall(){
consume(TokenType.IDENTIFIER,"Identifier expected");
if(check("=")){
match("=");
parseExpression();
} else if(check("(")){
// function call
advance(); // consume (
if(!check(")")) parseExpression(); // optional args
consume(TokenType.RPAREN,") expected");
} else if(check(".")){
advance();
consume(TokenType.IDENTIFIER,"Property expected");
}
// bare identifier — just consumed, do nothing
}


private void parseInput(){
match("लो");
consume(TokenType.IDENTIFIER,"Identifier expected after लो");
}


private void parseShow(){
match("दिखाओ");
parseExpression();
}


private void parseIf(){
if(check("यदि")||check("अगर")||check("अगर-अभिव्यक्ति"))
advance();

// FIX 8: Parse condition properly: left [relop right] [तब/तो]
parseExpression();
if(peek().type==TokenType.RELATIONAL){
advance();       // consume relational operator
parseExpression(); // consume right-hand side
}
if(check("तब")||check("तो"))
advance();

while(!check("अन्यथा")&&!check("नहीं तो")&&!check("समाप्तयदि")&&!check("अगर नहीं")&&!isEOF())
parseStatement();

if(check("अन्यथा")||check("नहीं तो")){
advance();
while(!check("समाप्तयदि")&&!check("अगर नहीं")&&!isEOF())
parseStatement();
}

if(check("समाप्तयदि")||check("अगर नहीं"))
advance();

}


private void parseWhile(){
advance();
parseExpression();
if(peek().type==TokenType.RELATIONAL){
advance();
parseExpression();
}
if(check("तब")||check("तो"))
advance();

while(!check("समाप्तजबतक")&&!check("समाप्त जब तक")&&!isEOF())
parseStatement();

advance();
}

private void parseForLoop(){
match("के लिए");
consume(TokenType.LPAREN,"( expected");

if(peek().type!=TokenType.RPAREN && !check(";"))
parseDeclaration();

consume(TokenType.SEMICOLON,"; expected");
parseExpression();
if(peek().type==TokenType.RELATIONAL){
advance();
parseExpression();
}
consume(TokenType.SEMICOLON,"; expected");

// increment expression: may be identifier = expr
if(peek().type==TokenType.IDENTIFIER){
advance(); // consume identifier
if(check("=")){ advance(); parseExpression(); }
}

consume(TokenType.RPAREN,") expected");

while(!check("समाप्तके लिए")&&!isEOF())
parseStatement();

advance(); // समाप्तके लिए
}

private void parseDoWhile(){
match("करो");
while(!check("जबतक")&&!check("जब तक")&&!isEOF())
parseStatement();
advance();
parseExpression();
}

private void parseSwitch(){
advance();
consume(TokenType.LPAREN,"( expected");
parseExpression();
consume(TokenType.RPAREN,") expected");
consume(TokenType.LBRACE,"{ expected");

while(!check("समाप्त स्विच")&&!check("समाप्तस्विच")&&!isEOF()){
if(check("मामला")){
advance();
parseExpression();
consume(TokenType.COLON,": expected");
}
else if(check("डिफ़ॉल्ट")){
advance();
consume(TokenType.COLON,": expected");
}
else parseStatement();
}
advance();
}


private void parseRepeat(){
match("दोहराओ");
consume(TokenType.NUMBER,"Number expected");
match("बार");
while(!check("समाप्तदोहराओ")&&!isEOF())
parseStatement();
match("समाप्तदोहराओ");
}


private void parseFunction(){
match("कार्य");
consume(TokenType.IDENTIFIER,"Function name expected");
if(check("(")){
advance();
// optional parameters
while(!check(")")&&!isEOF()){
if(peek().type==TokenType.IDENTIFIER) advance();
else if(check(",")) advance();
else break;
}
match(")");
}
while(!check("समाप्तकार्य")&&!isEOF())
parseStatement();
match("समाप्तकार्य");
}


private void parseReturn(){
advance();
parseExpression();
}

private void parseTryBlock(){
match("कोशिश");
consume(TokenType.LBRACE,"{ expected");
while(!check("पकड़ो")&&!check("अंत में")&&!check("}")&&!isEOF())
parseStatement();
consume(TokenType.RBRACE,"} expected");

if(check("पकड़ो")){
advance();
consume(TokenType.LPAREN,"( expected");
// exception variable declaration
advance(); // type
consume(TokenType.IDENTIFIER,"Exception var expected");
consume(TokenType.RPAREN,") expected");
consume(TokenType.LBRACE,"{ expected");
while(!check("अंत में")&&!check("}")&&!isEOF())
parseStatement();
consume(TokenType.RBRACE,"} expected");
}

if(check("अंत में")){
advance();
consume(TokenType.LBRACE,"{ expected");
while(!check("}")&&!isEOF())
parseStatement();
consume(TokenType.RBRACE,"} expected");
}
}

private void parseThrow(){
match("फेंको");
parseExpression();
}


private void parseClass(){
match("वर्ग");
consume(TokenType.IDENTIFIER,"Class name expected");
if(check("विस्तार करें")){
advance();
consume(TokenType.IDENTIFIER,"Parent class expected");
}
if(check("कार्यान्वयन करें")){
advance();
consume(TokenType.IDENTIFIER,"Interface expected");
}
if(check("{")){
advance();
}
while(!check("समाप्तवर्ग")&&!isEOF()){
if(check("निर्माता")) parseConstructor();
else parseStatement();
}
if(check("}")) advance();
match("समाप्तवर्ग");
}

private void parseInterface(){
match("अंतर्फलक");
consume(TokenType.IDENTIFIER,"Interface name expected");
if(check("विस्तार करें")){
advance();
consume(TokenType.IDENTIFIER,"Parent interface expected");
}
while(!check("समाप्त अंतर्फलक")&&!isEOF())
parseStatement();
advance();
}

// FIX 9: Record parser — params are plain identifiers, no full declarations needed
private void parseRecord(){
match("रिकॉर्ड");
consume(TokenType.IDENTIFIER,"Record name expected");
consume(TokenType.LPAREN,"( expected");
// Accept either plain identifiers or typed identifiers (मान id = 1 style)
while(!check(")")&&!isEOF()){
if(check("मान")||check("पूर्णांक")||check("दशमलव")||check("पाठ")){
advance(); // consume type keyword
consume(TokenType.IDENTIFIER,"Field name expected");
if(check("=")){ advance(); parseExpression(); } // optional default
} else if(peek().type==TokenType.IDENTIFIER){
advance();
} else {
advance();
}
if(check(",")) advance();
}
consume(TokenType.RPAREN,") expected");
while(!check("समाप्त रिकॉर्ड")&&!isEOF())
parseStatement();
advance();
}


private void parseConstructor(){
match("निर्माता");
while(!check("समाप्तनिर्माता")&&!isEOF())
parseStatement();
match("समाप्तनिर्माता");
}

private void parseAbstractFunction(){
match("अमूर्तकार्य");
consume(TokenType.IDENTIFIER,"Function name expected");
if(check("(")){
advance();
match(")");
}
}


private void parseAbstractClass(){
match("अमूर्तवर्ग");
consume(TokenType.IDENTIFIER,"Abstract class name expected");
if(check("विस्तार करें")){
advance();
consume(TokenType.IDENTIFIER,"Parent class expected");
}
while(!check("समाप्तअमूर्तवर्ग")&&!isEOF()){
if(check("अमूर्तकार्य")) parseAbstractFunction();
else parseStatement();
}
match("समाप्तअमूर्तवर्ग");
}


// FIX 10: parseExpression now only handles arithmetic; relational ops handled at statement level
private void parseExpression(){
parseTerm();
while(check("+")||check("-")||check("%")){
advance();
parseTerm();
}
}

private void parseTerm(){
parseFactor();
while(check("*")||check("/")){
advance();
parseFactor();
}
}

private void parseFactor(){
Token t=peek();

if(t.type==TokenType.NUMBER||
t.type==TokenType.FLOAT||
t.type==TokenType.STRING||
t.type==TokenType.IDENTIFIER||
t.value.equals("सत्य")||
t.value.equals("असत्य")){

advance();

if(check(".")){
match(".");
consume(TokenType.IDENTIFIER,"Property expected");
}
// FIX 11: Handle function call in expression context
if(check("(")){
advance();
if(!check(")")) parseExpression();
consume(TokenType.RPAREN,") expected");
}
}
else if(t.type==TokenType.LPAREN){
advance();
parseExpression();
consume(TokenType.RPAREN,"Missing )");
}
else{advance();}
}


private boolean check(String s){
return peek().value.equals(s);
}

private void match(String s){
if(check(s)) advance();
else throw error("Expected "+s);
}

private void consume(TokenType t,String msg){
if(peek().type==t) advance();
else throw error(msg);
}

private Token advance(){
return tokens.get(current++);
}

private Token peek(){
return current<tokens.size()?tokens.get(current):tokens.get(tokens.size()-1);
}

private boolean isEOF(){
return peek().type==TokenType.EOF;
}

private RuntimeException error(String m){
return new RuntimeException(m);
}

private void synchronize(){
advance();
while(!isEOF()){
if(check("समाप्त")||
check("मान")||
check("यदि")||
check("दिखाओ")||
check("कार्य"))
return;
advance();
}
}

}


// ================ INTERPRETER (Enhanced) =================

static class Interpreter {

private final List<Token> tokens;
private int current = 0;

private final Deque<Map<String, Object>> scopeStack = new ArrayDeque<>();
private final Map<String, int[]> functions = new HashMap<>();
private final Scanner inputScanner;

private Object returnValue = null;
private boolean returning = false;

Interpreter(List<Token> tokens, Scanner inputScanner) {
    this.tokens = tokens;
    this.inputScanner = inputScanner;
    pushScope();
}

private void pushScope() { scopeStack.push(new LinkedHashMap<>()); }
private void popScope()  { if (!scopeStack.isEmpty()) scopeStack.pop(); }

private void setVar(String name, Object value) {
    for (Map<String, Object> scope : scopeStack) {
        if (scope.containsKey(name)) { scope.put(name, value); return; }
    }
    scopeStack.peek().put(name, value);
}

private Object getVar(String name) {
    for (Map<String, Object> scope : scopeStack) {
        if (scope.containsKey(name)) return scope.get(name);
    }
    throw new RuntimeException("Undefined variable: " + name);
}

private Token peek() {
    return current < tokens.size() ? tokens.get(current) : tokens.get(tokens.size() - 1);
}

private Token advance() { return tokens.get(current++); }

private boolean check(String s) { return peek().value.equals(s); }
private boolean checkType(TokenType t) { return peek().type == t; }

private void match(String s) {
    if (check(s)) advance();
    else throw new RuntimeException("Expected '" + s + "' but got '" + peek().value + "' at line " + peek().line);
}

private boolean isEOF() { return peek().type == TokenType.EOF; }

void execute() {
    match("शुरू");
    while (!check("समाप्त") && !isEOF()) {
        executeStatement();
        if (returning) break;
    }
    match("समाप्त");
}

private void executeStatement() {
    String v = peek().value;
    TokenType t = peek().type;

    if (v.equals("मान") || v.equals("पूर्णांक") || v.equals("दशमलव")
            || v.equals("पाठ") || v.equals("बूलियन") || v.equals("पूर्ण संख्या")
            || v.equals("दशमलव संख्या") || v.equals("वर्ण") || v.equals("लंबी")
            || v.equals("छोटी") || v.equals("बहुत छोटी") || v.equals("असली")
            || v.equals("वस्तु") || v.equals("var") || v.equals("सर्वनाम")) {
        executeDeclaration();
    } else if (v.equals("दिखाओ")) {
        executeShow();
    } else if (v.equals("लो")) {
        executeInput();
    } else if (v.equals("यदि") || v.equals("अगर") || v.equals("अगर-अभिव्यक्ति")) {
        executeIf();
    } else if (v.equals("जबतक") || v.equals("जब तक")) {
        executeWhile();
    } else if (v.equals("दोहराओ")) {
        executeRepeat();
    } else if (v.equals("के लिए")) {
        executeForLoop();
    } else if (v.equals("करो")) {
        executeDoWhile();
    } else if (v.equals("स्विच") || v.equals("स्विच-अभिव्यक्ति")) {
        executeSwitch();
    } else if (v.equals("कार्य")) {
        registerFunction();
    } else if (v.equals("वापस") || v.equals("वापसी")) {
        executeReturn();
    } else if (v.equals("कोशिश")) {
        executeTryBlock();
    } else if (v.equals("फेंको")) {
        executeThrow();
    } else if (v.equals("वर्ग") || v.equals("अमूर्तवर्ग") || v.equals("अंतर्फलक") || v.equals("रिकॉर्ड")) {
        skipClassBlock();
    } else if (t == TokenType.IDENTIFIER) {
        executeAssignmentOrCall();
    } else {
        advance();
    }
}

private void executeDeclaration() {
    advance();
    String name = peek().value;
    advance();
    match("=");
    Object value = evalExpression();
    setVar(name, value);
}

// FIX 12: Interpreter also handles function calls (identifier followed by '()')
private void executeAssignmentOrCall() {
    String name = peek().value;
    advance();

    if (check("=")) {
        advance();
        Object value = evalExpression();
        setVar(name, value);
    } else if (check("(")) {
        advance(); // consume (
        // optional arguments (ignored in this interpreter)
        if (!check(")")) evalExpression();
        match(")");
        if (functions.containsKey(name)) {
            callFunction(name);
        }
    } else if (check(".")) {
        advance();
        advance(); // property name
    }
    // bare identifier — skip
}

private void executeShow() {
    match("दिखाओ");
    Object value = evalExpression();
    System.out.println(formatValue(value));
}

private void executeInput() {
    match("लो");
    String name = peek().value;
    advance();
    System.out.print("Input for " + name + ": ");
    String input = inputScanner.nextLine().trim();
    try {
        if (input.contains(".")) setVar(name, Double.parseDouble(input));
        else setVar(name, Double.parseDouble(input));
    } catch (NumberFormatException e) {
        setVar(name, input);
    }
}

// FIX 13: executeIf properly evaluates condition with relational operator
private void executeIf() {
    advance(); // consume यदि/अगर
    Object left = evalExpression();

    boolean condition;
    if (peek().type == TokenType.RELATIONAL) {
        String op = peek().value;
        advance();
        Object right = evalExpression();
        condition = evalRelation(left, op, right);
    } else {
        condition = isTruthy(left);
    }

    if (check("तब") || check("तो")) advance();

    if (condition) {
        while (!check("अन्यथा") && !check("नहीं तो") && !check("समाप्तयदि") && !check("अगर नहीं") && !isEOF()) {
            executeStatement();
            if (returning) break;
        }
        if (check("अन्यथा") || check("नहीं तो")) {
            advance();
            skipUntil("समाप्तयदि", "अगर नहीं");
        }
    } else {
        skipUntil("अन्यथा", "नहीं तो", "समाप्तयदि", "अगर नहीं");
        if (check("अन्यथा") || check("नहीं तो")) {
            advance();
            while (!check("समाप्तयदि") && !check("अगर नहीं") && !isEOF()) {
                executeStatement();
                if (returning) break;
            }
        }
    }
    if (check("समाप्तयदि") || check("अगर नहीं")) advance();
}

private void executeWhile() {
    int loopStart = current;

    while (true) {
        current = loopStart;
        advance(); // consume जबतक

        Object left = evalExpression();
        boolean cond;

        if (peek().type == TokenType.RELATIONAL) {
            String op = peek().value;
            advance();
            Object right = evalExpression();
            cond = evalRelation(left, op, right);
        } else {
            cond = isTruthy(left);
        }

        if (check("तब") || check("तो")) advance();

        if (!cond) {
            skipUntil("समाप्तजबतक", "समाप्त जब तक");
            if (check("समाप्तजबतक") || check("समाप्त जब तक")) advance();
            break;
        }

        while (!check("समाप्तजबतक") && !check("समाप्त जब तक") && !isEOF()) {
            executeStatement();
            if (returning) break;
        }
        if (check("समाप्तजबतक") || check("समाप्त जब तक")) advance();
        if (returning) break;
    }
}

private void executeRepeat() {
    match("दोहराओ");
    int times = (int) Double.parseDouble(peek().value);
    advance();
    match("बार");

    int bodyStart = current;

    for (int i = 0; i < times; i++) {
        current = bodyStart;
        while (!check("समाप्तदोहराओ") && !isEOF()) {
            executeStatement();
            if (returning) break;
        }
        if (returning) break;
    }

    if (!check("समाप्तदोहराओ")) skipUntil("समाप्तदोहराओ");
    match("समाप्तदोहराओ");
}

private void executeForLoop() {
    match("के लिए");
    match("(");

    if (!check(";")) executeDeclaration();
    match(";");

    int condStart = current;

    // evaluate condition
    Object leftC = evalExpression();
    String condOp = null;
    Object rightC = null;
    if (peek().type == TokenType.RELATIONAL) {
        condOp = peek().value;
        advance();
        rightC = evalExpression();
    }
    match(";");

    int incrStart = current;
    // parse (but don't execute yet) the increment
    String incrVar = peek().type == TokenType.IDENTIFIER ? peek().value : null;
    if (incrVar != null) { advance(); if (check("=")) { advance(); evalExpression(); } }
    match(")");

    int bodyStart = current;

    boolean cond = condOp != null ? evalRelation(leftC, condOp, rightC) : isTruthy(leftC);

    while (cond) {
        current = bodyStart;
        while (!check("समाप्तके लिए") && !isEOF()) {
            executeStatement();
            if (returning) break;
        }
        if (returning) break;

        // execute increment
        if (incrVar != null) {
            current = incrStart;
            advance(); // var name
            if (check("=")) { advance(); Object v = evalExpression(); setVar(incrVar, v); }
        }

        // re-evaluate condition
        current = condStart;
        Object lc = evalExpression();
        if (peek().type == TokenType.RELATIONAL) {
            String op2 = peek().value; advance();
            Object rc = evalExpression();
            cond = evalRelation(lc, op2, rc);
        } else {
            cond = isTruthy(lc);
        }
    }

    skipUntil("समाप्तके लिए");
    if (check("समाप्तके लिए")) advance();
}

private void executeDoWhile() {
    match("करो");
    int bodyStart = current;

    do {
        current = bodyStart;
        while (!check("जबतक") && !check("जब तक") && !isEOF()) {
            executeStatement();
            if (returning) break;
        }
        advance(); // consume जबतक/जब तक
        Object cond = evalExpression();
        if (!isTruthy(cond)) break;
    } while (true);
}

private void executeSwitch() {
    match("स्विच");
    match("(");
    Object switchValue = evalExpression();
    match(")");
    match("{");

    while (!check("समाप्त स्विच") && !check("समाप्तस्विच") && !isEOF()) {
        if (check("मामला")) {
            advance();
            Object caseValue = evalExpression();
            match(":");
            if (switchValue.equals(caseValue)) {
                while (!check("मामला") && !check("डिफ़ॉल्ट") && !check("समाप्त स्विच") && !check("समाप्तस्विच") && !isEOF()) {
                    executeStatement();
                    if (returning) break;
                }
            } else {
                skipUntil("मामला", "डिफ़ॉल्ट", "समाप्त स्विच", "समाप्तस्विच");
            }
        } else if (check("डिफ़ॉल्ट")) {
            advance();
            match(":");
            while (!check("मामला") && !check("समाप्त स्विच") && !check("समाप्तस्विच") && !isEOF()) {
                executeStatement();
            }
        } else {
            advance();
        }
    }
    if (check("समाप्त स्विच") || check("समाप्तस्विच")) advance();
}

private void registerFunction() {
    match("कार्य");
    String name = peek().value;
    advance();

    if (check("(")) {
        advance();
        while (!check(")") && !isEOF()) advance();
        match(")");
    }

    int bodyStart = current;
    skipUntil("समाप्तकार्य");
    int bodyEnd = current;
    match("समाप्तकार्य");

    functions.put(name, new int[]{bodyStart, bodyEnd});
}

private void executeReturn() {
    advance();
    returnValue = evalExpression();
    returning = true;
}

private void executeTryBlock() {
    match("कोशिश");
    match("{");
    while (!check("पकड़ो") && !check("अंत में") && !check("}") && !isEOF()) {
        executeStatement();
    }
    match("}");

    if (check("पकड़ो")) {
        advance();
        match("(");
        advance(); // type keyword
        String exVar = peek().value; advance();
        match(")");
        match("{");
        while (!check("अंत में") && !check("}") && !isEOF()) executeStatement();
        match("}");
    }

    if (check("अंत में")) {
        advance();
        match("{");
        while (!check("}") && !isEOF()) executeStatement();
        match("}");
    }
}

private void executeThrow() {
    match("फेंको");
    Object exception = evalExpression();
    throw new RuntimeException("Exception: " + formatValue(exception));
}

private Object callFunction(String name) {
    if (!functions.containsKey(name)) {
        throw new RuntimeException("Undefined function: " + name);
    }
    int[] range = functions.get(name);
    int savedPos = current;
    Object savedReturn = returnValue;
    boolean savedReturning = returning;

    current = range[0];
    returning = false;
    returnValue = null;

    pushScope();
    while (current < range[1] && !check("समाप्तकार्य") && !isEOF()) {
        executeStatement();
        if (returning) break;
    }
    popScope();

    Object result = returnValue;
    current = savedPos;
    returnValue = savedReturn;
    returning = savedReturning;

    return result;
}

private void skipClassBlock() {
    String keyword = peek().value;
    advance();
    advance(); // class name

    String endKeyword = "समाप्तवर्ग";
    if (keyword.equals("अमूर्तवर्ग")) endKeyword = "समाप्तअमूर्तवर्ग";
    else if (keyword.equals("अंतर्फलक")) endKeyword = "समाप्त अंतर्फलक";
    else if (keyword.equals("रिकॉर्ड")) endKeyword = "समाप्त रिकॉर्ड";

    skipUntil(endKeyword);
    if (check(endKeyword)) advance();
}

// ---- Expression evaluator ----

private Object evalExpression() {
    Object left = evalTerm();
    while (check("+") || check("-") || check("%")) {
        String op = peek().value; advance();
        Object right = evalTerm();
        if (op.equals("+")) {
            if (left instanceof String || right instanceof String)
                left = formatValue(left) + formatValue(right);
            else
                left = toDouble(left) + toDouble(right);
        } else if (op.equals("-")) {
            left = toDouble(left) - toDouble(right);
        } else if (op.equals("%")) {
            left = toDouble(left) % toDouble(right);
        }
    }
    return left;
}

private Object evalTerm() {
    Object left = evalFactor();
    while (check("*") || check("/")) {
        String op = peek().value; advance();
        Object right = evalFactor();
        if (op.equals("*")) left = toDouble(left) * toDouble(right);
        else left = toDouble(left) / toDouble(right);
    }
    return left;
}

private Object evalFactor() {
    Token t = peek();

    if (t.type == TokenType.NUMBER || t.type == TokenType.FLOAT) {
        advance(); return Double.parseDouble(t.value);
    }
    if (t.type == TokenType.STRING) { advance(); return t.value; }
    if (t.value.equals("सत्य"))  { advance(); return Boolean.TRUE; }
    if (t.value.equals("असत्य")) { advance(); return Boolean.FALSE; }

    if (t.type == TokenType.IDENTIFIER) {
        advance();
        String name = t.value;

        if (check(".")) {
            advance();
            String prop = peek().value; advance();
            Object obj = getVar(name);
            if (prop.equals("लंबाई") || prop.equals("length"))
                return (double) formatValue(obj).length();
            return null;
        }

        // FIX 14: Function call in expression context
        if (check("(")) {
            advance();
            if (!check(")")) evalExpression(); // optional args
            match(")");
            if (functions.containsKey(name)) return callFunction(name);
            return null;
        }

        return getVar(name);
    }

    if (t.type == TokenType.LPAREN) {
        advance();
        Object val = evalExpression();
        match(")");
        return val;
    }

    // FIX 15: Don't throw on keyword tokens encountered in expression position (e.g. EOF sentinel)
    advance();
    return null;
}

private boolean evalRelation(Object left, String op, Object right) {
    if (left instanceof String || right instanceof String) {
        String l = formatValue(left);
        String r = formatValue(right);
        switch (op) {
            case "==": return l.equals(r);
            case "!=": return !l.equals(r);
        }
    }
    double l = toDouble(left);
    double r = toDouble(right);
    switch (op) {
        case "==": return l == r;
        case "!=": return l != r;
        case "<":  return l < r;
        case ">":  return l > r;
        case "<=": return l <= r;
        case ">=": return l >= r;
    }
    return false;
}

private boolean isTruthy(Object val) {
    if (val == null) return false;
    if (val instanceof Boolean) return (Boolean) val;
    if (val instanceof Double) return ((Double) val) != 0;
    if (val instanceof String) return !((String) val).isEmpty();
    return true;
}

private double toDouble(Object val) {
    if (val instanceof Double) return (Double) val;
    if (val instanceof Boolean) return ((Boolean) val) ? 1 : 0;
    if (val instanceof String) {
        try { return Double.parseDouble((String) val); }
        catch (NumberFormatException e) { return 0; }
    }
    return 0;
}

private String formatValue(Object val) {
    if (val == null) return "null";
    if (val instanceof Double) {
        double d = (Double) val;
        if (d == Math.floor(d) && !Double.isInfinite(d))
            return String.valueOf((long) d);
        return String.valueOf(d);
    }
    return val.toString();
}

private void skipUntil(String... keywords) {
    Set<String> stopAt = new HashSet<>(Arrays.asList(keywords));
    int depth = 0;
    while (!isEOF()) {
        String v = peek().value;
        if (v.equals("यदि") || v.equals("जबतक") || v.equals("दोहराओ")
                || v.equals("कार्य") || v.equals("वर्ग") || v.equals("अंतर्फलक")
                || v.equals("अमूर्तवर्ग") || v.equals("के लिए") || v.equals("करो")
                || v.equals("कोशिश")) {
            depth++;
        }
        if (stopAt.contains(v)) {
            if (depth == 0) return;
            if (v.startsWith("समाप्त") || v.startsWith("अंत")) depth--;
        }
        advance();
    }
}

}

//======================================================
//RESEARCH PAPER WORKING
//Inspired by Explain in Plain Language (EiPL)
//======================================================

static void researchPaperWorking(String sourceCode){

 System.out.println("\n====================================");
 System.out.println("RESEARCH PAPER WORKING");
 System.out.println("====================================");

 // ----------------------------------
 // 1. Code Explanation
 // ----------------------------------

 if(sourceCode.contains("यदि")){
     System.out.println(
     "✓ Code Explanation:");
     System.out.println(
     "यह प्रोग्राम निर्णय (Decision Making) करता है।");
 }

 if(sourceCode.contains("अन्यथा")){
     System.out.println(
     "✓ Code Explanation:");
     System.out.println(
     "यह प्रोग्राम Else (अन्यथा) शर्त को भी संभालता है।");
 }

 if(sourceCode.contains("जबतक")){
     System.out.println(
     "✓ Code Explanation:");
     System.out.println(
     "यह प्रोग्राम Loop का उपयोग करता है।");
 }

 if(sourceCode.contains("दोहराओ")){
     System.out.println(
     "✓ Code Explanation:");
     System.out.println(
     "यह प्रोग्राम किसी क्रिया को बार-बार (Repeat) दोहराता है।");
 }

 if(sourceCode.contains("लो")){
     System.out.println(
     "✓ Code Explanation:");
     System.out.println(
     "यह प्रोग्राम यूजर से Input लेता है।");
 }

 if(sourceCode.contains("दिखाओ")){
     System.out.println(
     "✓ Code Explanation:");
     System.out.println(
     "यह प्रोग्राम Output प्रदर्शित करता है।");
 }

 // ----------------------------------
 // 2. Hinglish Detection
 // ----------------------------------

 boolean hindiFound =
     sourceCode.contains("यदि")
     || sourceCode.contains("दिखाओ")
     || sourceCode.contains("मान")
     || sourceCode.contains("लो")
     || sourceCode.contains("दोहराओ");

 boolean englishFound =
     sourceCode.contains("show")
     || sourceCode.contains("print")
     || sourceCode.contains("if")
     || sourceCode.contains("input")
     || sourceCode.contains("loop")
     || sourceCode.contains("while")
     || sourceCode.contains("for");

 if(hindiFound && englishFound){

     System.out.println(
     "\n✓ Hinglish Program Detected");

 }

 // ----------------------------------
 // 3. Error Suggestions
 // ----------------------------------

 if(sourceCode.contains("यदि")
 && !sourceCode.contains("समाप्तयदि")){

     System.out.println(
     "\n⚠ Suggestion:");
     System.out.println(
     "'समाप्तयदि' नहीं मिला।");

 }

 if(sourceCode.contains("जबतक")
 && !sourceCode.contains("समाप्तजबतक")){

     System.out.println(
     "\n⚠ Suggestion:");
     System.out.println(
     "'समाप्तजबतक' नहीं मिला।");

 }

 if(sourceCode.contains("दोहराओ")
 && !sourceCode.contains("समाप्तदोहराओ")){

     System.out.println(
     "\n⚠ Suggestion:");
     System.out.println(
     "'समाप्तदोहराओ' नहीं मिला।");

 }

 // ----------------------------------
 // 4. Variable Analysis
 // ----------------------------------

 int variables = 0;

 String[] lines = sourceCode.split("\n");

 for(String line : lines){

     line = line.trim();

     if(line.startsWith("मान ") || line.startsWith("लो "))
         variables++;
 }

 System.out.println(
 "\n✓ Variables Found : "
 + variables);

 // ----------------------------------
 // 5. Natural Language Detection
 // ----------------------------------

 if(sourceCode.contains("1 से 10 तक संख्या दिखाओ")){

     System.out.println(
     "\n✓ Natural Language Request Detected");

     System.out.println(
     "\nGenerated Hindi Program:\n");

     System.out.println(
     "मान i = 1\n"+
     "जबतक i <= 10 तब\n"+
     "दिखाओ i\n"+
     "i = i + 1\n"+
     "समाप्तजबतक");
 }

 // ----------------------------------
 // 6. Research Statistics
 // ----------------------------------

 int keywords = 0;

 String[] researchKeywords = {
 "मान",
 "यदि",
 "अन्यथा",
 "जबतक",
 "दिखाओ",
 "कार्य",
 "वर्ग",
 "लो",
 "दोहराओ"
 };

 for(String k : researchKeywords){

     if(sourceCode.contains(k))
         keywords++;
 }

 System.out.println(
 "\n✓ Hindi Keywords Used : "
 + keywords);

 // ----------------------------------
 // 7. Final Research Result
 // ----------------------------------

 System.out.println(
 "\n✓ Research Paper Features Active");

 System.out.println(
 "   • Explain in Plain Language");

 System.out.println(
 "   • Hinglish Detection");

 System.out.println(
 "   • Error Suggestions");

 System.out.println(
 "   • Natural Language Support");

 System.out.println(
 "   • Program Analysis");

 System.out.println(
 "\n====================================");
}



static String generateHindiCode(String desc) {

    desc = desc.toLowerCase();

    // Normalize spacing
    desc = desc.replaceAll("\\s+", " ");

    // Pull out any numbers in the description — used to size loop ranges
    // and repeat counts instead of always hardcoding 1-10 / 3 times.
    // (Manual character-by-character scan instead of java.util.regex,
    // preserving the exact same extraction order/behavior as before.)
    List<Integer> numbers = new ArrayList<>();
    int scanIdx = 0;
    while (scanIdx < desc.length()) {
        char c = desc.charAt(scanIdx);
        if (Character.isDigit(c)) {
            int digitStart = scanIdx;
            while (scanIdx < desc.length() && Character.isDigit(desc.charAt(scanIdx))) {
                scanIdx++;
            }
            numbers.add(Integer.parseInt(desc.substring(digitStart, scanIdx)));
        } else {
            scanIdx++;
        }
    }

    // -------------------------------
    // 1. INPUT PROGRAMS (checked FIRST so "यूजर...लो...जोड़ो" doesn't
    //    get stolen by the arithmetic check further down)
    // -------------------------------
    boolean wantsInput =
            (desc.contains("यूजर") && desc.contains("लो"))
            || desc.contains("input");

    if (wantsInput) {
        return "शुरू\n" +
               "लो a\n" +
               "लो b\n" +
               "मान result = a + b\n" +
               "दिखाओ result\n" +
               "समाप्त";
    }

    // -------------------------------
    // 2. EVEN / FILTER LOOP (checked before the generic loop, since
    //    "even" alone is enough regardless of how the range is phrased)
    // -------------------------------
    if (desc.contains("even") || desc.contains("सम संख्या") || desc.contains("सम नंबर")) {
        int start = numbers.size() >= 2 ? numbers.get(0) : 1;
        int end   = numbers.size() >= 2 ? numbers.get(1)
                  : numbers.size() == 1 ? numbers.get(0) : 10;
        return "शुरू\n" +
               "मान i = " + start + "\n" +
               "जबतक i <= " + end + " तब\n" +
               "यदि i % 2 == 0 तब\n" +
               "दिखाओ i\n" +
               "समाप्तयदि\n" +
               "i = i + 1\n" +
               "समाप्तजबतक\n" +
               "समाप्त";
    }

    // -------------------------------
    // 3. GENERIC RANGE LOOP ("1 से 10 तक", "loop from 1 to 10",
    //    "print numbers from 1 to 10")
    // -------------------------------
    boolean wantsLoop =
            (desc.contains("से") && desc.contains("तक"))
            || desc.contains("loop")
            || desc.matches(".*\\bto\\b.*")
            || desc.contains("तक");

    if (wantsLoop) {
        int start = numbers.size() >= 2 ? numbers.get(0) : 1;
        int end   = numbers.size() >= 2 ? numbers.get(1)
                  : numbers.size() == 1 ? numbers.get(0) : 10;
        return "शुरू\n" +
               "मान i = " + start + "\n" +
               "जबतक i <= " + end + " तब\n" +
               "दिखाओ i\n" +
               "i = i + 1\n" +
               "समाप्तजबतक\n" +
               "समाप्त";
    }

    // -------------------------------
    // 4. REPEAT LOOP ("3 बार", "5 times", "repeat ... times")
    // -------------------------------
    if (desc.contains("बार") || desc.contains("times") || desc.contains("repeat")) {
        int times = numbers.isEmpty() ? 3 : numbers.get(0);
        return "शुरू\n" +
               "दोहराओ " + times + " बार\n" +
               "दिखाओ \"hello\"\n" +
               "समाप्तदोहराओ\n" +
               "समाप्त";
    }

    // -------------------------------
    // 5. IF / COMPARISON CONDITIONS
    // -------------------------------
    boolean wantsIf =
            desc.contains("अगर") || desc.contains("if")
            || desc.contains("compare") || desc.contains("बड़ा")
            || desc.contains("bigger") || desc.contains("बराबर")
            || desc.contains("equal");

    if (wantsIf) {

        if (desc.contains("बराबर") || desc.contains("equal")) {
            return "शुरू\n" +
                   "यदि a == b तब\n" +
                   "दिखाओ \"equal\"\n" +
                   "समाप्तयदि\n" +
                   "समाप्त";
        }

        if (desc.contains("बड़ा") || desc.contains("bigger") || desc.contains("compare")
                || desc.contains("else") || desc.contains("वरना") || desc.contains("नहीं तो")) {
            return "शुरू\n" +
                   "यदि a > b तब\n" +
                   "दिखाओ a\n" +
                   "अन्यथा\n" +
                   "दिखाओ b\n" +
                   "समाप्तयदि\n" +
                   "समाप्त";
        }

        return "शुरू\n" +
               "यदि a > b तब\n" +
               "दिखाओ a\n" +
               "समाप्तयदि\n" +
               "समाप्त";
    }

    // -------------------------------
    // 6. MAX / MIN LOGIC
    // -------------------------------
    if (desc.contains("maximum") || desc.contains("max")
            || desc.contains("largest") || desc.contains("biggest")) {
        return "शुरू\n" +
               "यदि a > b तब\n" +
               "दिखाओ a\n" +
               "अन्यथा\n" +
               "दिखाओ b\n" +
               "समाप्तयदि\n" +
               "समाप्त";
    }

    if (desc.contains("minimum") || desc.contains("min") || desc.contains("smallest")) {
        return "शुरू\n" +
               "यदि a < b तब\n" +
               "दिखाओ a\n" +
               "अन्यथा\n" +
               "दिखाओ b\n" +
               "समाप्तयदि\n" +
               "समाप्त";
    }

    // -------------------------------
    // 7. BASIC ARITHMETIC
    // -------------------------------
    if (desc.contains("जोड़") || desc.contains("add") || desc.contains("plus")) {
        return "शुरू\n" +
               "मान a = 10\n" +
               "मान b = 20\n" +
               "मान result = a + b\n" +
               "दिखाओ result\n" +
               "समाप्त";
    }

    if (desc.contains("घटा") || desc.contains("subtract") || desc.contains("minus")) {
        return "शुरू\n" +
               "मान a = 10\n" +
               "मान b = 5\n" +
               "मान result = a - b\n" +
               "दिखाओ result\n" +
               "समाप्त";
    }

    if (desc.contains("गुणा") || desc.contains("multiply")) {
        return "शुरू\n" +
               "मान a = 5\n" +
               "मान b = 5\n" +
               "मान result = a * b\n" +
               "दिखाओ result\n" +
               "समाप्त";
    }

    if (desc.contains("भाग") || desc.contains("divide")) {
        return "शुरू\n" +
               "मान a = 10\n" +
               "मान b = 2\n" +
               "मान result = a / b\n" +
               "दिखाओ result\n" +
               "समाप्त";
    }

    // -------------------------------
    // 8. SIMPLE OUTPUT ("hello world", "print hello", "display message")
    // -------------------------------
    if (desc.contains("hello") || desc.contains("world") || desc.contains("display")
            || desc.contains("message") || desc.contains("print")) {
        String msg;
        if (desc.contains("hello") && desc.contains("world")) msg = "hello world";
        else if (desc.contains("hello")) msg = "hello";
        else if (desc.contains("message")) msg = "message";
        else msg = "hello world";

        return "शुरू\n" +
               "दिखाओ \"" + msg + "\"\n" +
               "समाप्त";
    }

    // -------------------------------
    // 9. DEFAULT FALLBACK
    // -------------------------------
    return "शुरू\n" +
           "// समझ नहीं आया description\n" +
           "दिखाओ \"invalid input\"\n" +
           "समाप्त";
}
//================ MAIN =================


public static void main(String[] args){

    Scanner sc = new Scanner(System.in);

    System.out.println("Choose mode:");
    System.out.println("1. Hindi Code Input");
    System.out.println("2. Description → Hindi Code");

    String mode = sc.nextLine();

    String program;

    if(mode.equals("2")){

        System.out.println("Enter simple description:");
        String desc = sc.nextLine();

        program = generateHindiCode(desc);

        System.out.println("\n=========== GENERATED HINDI CODE ===========");
        System.out.println(program);
    }
    else{

        System.out.println("Enter Hindi Language Program:");

        StringBuilder sb = new StringBuilder();
        String line;

        while(sc.hasNextLine()){
            line = sc.nextLine();
            sb.append(line).append("\n");

            if(line.trim().equals("समाप्त"))
                break;
        }

        program = sb.toString();
    }

    System.out.println("\n=========== SOURCE CODE ===========");
    System.out.println(program);

    // SAME PIPELINE FOR BOTH MODES
    researchPaperWorking(program);

    System.out.println("\n=========== TOKENS ===========");

    Lexer lexer = new Lexer(program);
    List<Token> tokens = lexer.tokenize();

    for(Token t : tokens)
        System.out.println(t);

    System.out.println("\n=========== PARSER ===========");

    Parser parser = new Parser(tokens);
    parser.parseProgram();

    System.out.println("\n=========== OUTPUT ===========");

    Interpreter interpreter =
            new Interpreter(tokens, sc);

    try{
        interpreter.execute();
    }catch(Exception e){
        System.out.println("Runtime Error: " + e.getMessage());
    }

    sc.close();
}

}
