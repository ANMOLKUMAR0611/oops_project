#include <iostream>
#include <fstream>
#include <string>
#include <iomanip>
#include <ctime>
using namespace std;

// Base class
class SubscriptionPlan{
public:
    virtual string getTierName()=0;
    virtual double getMonthlyRate()=0;
    virtual int getMaxDevices()=0;
    virtual string getFeatures()=0;
    virtual int getGraceDays(){
        return 0;
    }
    virtual double calculatePrice(int months){
        double total=getMonthlyRate()*months;
        if(months==3){
            total=total*0.90;
        }
        if(months==12){
            total=total*0.80;
        }
        return total;
    }
    virtual ~SubscriptionPlan(){}
};

// Derived classes
class NormalPlan:public SubscriptionPlan{
public:
    string getTierName(){ return "Normal"; }
    double getMonthlyRate(){ return 199; }
    int getMaxDevices(){ return 1; }
    string getFeatures(){ return "Basic App Access (Standard Quality)"; }
};

class SilverPlan:public SubscriptionPlan{
public:
    string getTierName(){ return "Silver"; }
    double getMonthlyRate(){ return 499; }
    int getMaxDevices(){ return 2; }
    string getFeatures(){ return "Standard + Ad-Free + HD Access"; }
    int getGraceDays(){ return 3; }
};

class PremiumPlan:public SubscriptionPlan{
public:
    string getTierName(){ return "Premium"; }
    double getMonthlyRate(){ return 999; }
    int getMaxDevices(){ return 5; }
    string getFeatures(){ return "All Features + Priority Support + Offline"; }
    int getGraceDays(){ return 7; }
    double calculatePrice(int months){
        double total=getMonthlyRate()*months;
        if(months==3){
            total=total*0.90;
        }
        if(months==12){
            total=total*0.75;
        }
        return total;
    }
};

SubscriptionPlan* makePlan(string tier){
    if(tier=="Silver") return new SilverPlan();
    if(tier=="Premium") return new PremiumPlan();
    return new NormalPlan();
}

string formatDate(time_t t){
    if(t==0) return "Never";
    tm* ltm=localtime(&t);
    string d=to_string(ltm->tm_mday);
    string m=to_string(1+ltm->tm_mon);
    string y=to_string(1900+ltm->tm_year);
    if(d.length()==1) d="0"+d;
    if(m.length()==1) m="0"+m;
    return d+"-"+m+"-"+y;
}

time_t addMonths(time_t start,int months){
    tm temp=*localtime(&start);
    temp.tm_mon=temp.tm_mon+months;
    return mktime(&temp);
}

int daysLeft(time_t due){
    double sec=difftime(due,time(0));
    return (int)(sec/86400);
}

class Subscriber{
public:
    int userId;
    string name;
    string email;
    string mobile;
    SubscriptionPlan* plan;
    int durationMonths;
    double amountPaid;
    time_t startDate;
    time_t dueDate;
    bool isActive;

    Subscriber(){
        userId=0;
        plan=new NormalPlan();
        durationMonths=0;
        amountPaid=0;
        startDate=0;
        dueDate=0;
        isActive=false;
    }

    bool hasValidAccess(){
        if(!isActive) return false;
        time_t now=time(0);
        time_t graceEnd=dueDate+(plan->getGraceDays()*86400);
        if(difftime(graceEnd,now)>=0) return true;
        return false;
    }

    string getStatus(){
        if(!isActive) return "Revoked";
        time_t now=time(0);
        if(difftime(dueDate,now)>=0){
            if(daysLeft(dueDate)<=7) return "Expiring Soon";
            return "Active";
        }
        time_t graceEnd=dueDate+(plan->getGraceDays()*86400);
        if(difftime(graceEnd,now)>=0) return "Grace Period";
        return "Expired";
    }
};

class SubscriptionManager{
private:
    Subscriber list[100];
    int count;
    int nextId;

public:
    SubscriptionManager(){
        count=0;
        nextId=1001;
        loadFromFile();
    }

    void saveToFile(){
        ofstream out("subscribers.txt");
        for(int i=0;i<count;i++){
            out<<list[i].userId<<"|"
               <<list[i].name<<"|"
               <<list[i].email<<"|"
               <<list[i].mobile<<"|"
               <<list[i].plan->getTierName()<<"|"
               <<list[i].durationMonths<<"|"
               <<list[i].amountPaid<<"|"
               <<list[i].startDate<<"|"
               <<list[i].dueDate<<"|"
               <<list[i].isActive<<"\n";
        }
        out.close();
    }

    void loadFromFile(){
        ifstream in("subscribers.txt");
        if(!in) return;
        count=0;
        string temp;
        while(getline(in,temp,'|')){
            if(temp=="") break;
            list[count].userId=stoi(temp);
            getline(in,list[count].name,'|');
            getline(in,list[count].email,'|');
            getline(in,list[count].mobile,'|');
            string tier;
            getline(in,tier,'|');
            list[count].plan=makePlan(tier);
            getline(in,temp,'|');
            list[count].durationMonths=stoi(temp);
            getline(in,temp,'|');
            list[count].amountPaid=stod(temp);
            getline(in,temp,'|');
            list[count].startDate=stoll(temp);
            getline(in,temp,'|');
            list[count].dueDate=stoll(temp);
            getline(in,temp);
            list[count].isActive=(temp=="1");

            if(list[count].userId>=nextId){
                nextId=list[count].userId+1;
            }
            count++;
        }
        in.close();
    }

    SubscriptionPlan* selectPlan(){
        int ch;
        cout<<"\n1. Normal (Rs. 199/mo | 1 Device | 0d Grace)\n";
        cout<<"2. Silver (Rs. 499/mo | 2 Devices | 3d Grace)\n";
        cout<<"3. Premium (Rs. 999/mo | 5 Devices | 7d Grace)\n";
        cout<<"Choose Tier (1-3): ";
        cin>>ch;
        if(ch==2) return new SilverPlan();
        if(ch==3) return new PremiumPlan();
        return new NormalPlan();
    }

    int selectDuration(){
        int ch;
        cout<<"\n1. 1 Month\n";
        cout<<"2. 3 Months (10% Off)\n";
        cout<<"3. 1 Year (20% Off | 25% Off for Premium)\n";
        cout<<"Choose Duration (1-3): ";
        cin>>ch;
        if(ch==2) return 3;
        if(ch==3) return 12;
        return 1;
    }

    double applyPromo(double amt,int months,string &usedCode){
        string code;
        cout<<"Enter Promo Code (or NONE): ";
        cin>>code;
        if(code=="WELCOME10"){
            usedCode="WELCOME10 (10% Off)";
            return amt*0.90;
        }
        if(code=="STUDENT20"){
            usedCode="STUDENT20 (20% Off)";
            return amt*0.80;
        }
        if(code=="ANNUAL25" && months==12){
            usedCode="ANNUAL25 (25% Off)";
            return amt*0.75;
        }
        usedCode="NONE";
        return amt;
    }

    void printReceipt(Subscriber s,double paid,string promo,string type){
        cout<<"\n-----------------------------------------\n";
        cout<<"             PAYMENT RECEIPT             \n";
        cout<<"-----------------------------------------\n";
        cout<<"Type       : "<<type<<"\n";
        cout<<"User ID    : "<<s.userId<<"\n";
        cout<<"Name       : "<<s.name<<"\n";
        cout<<"Plan       : "<<s.plan->getTierName()<<" ("<<s.durationMonths<<" Months)\n";
        cout<<"Promo Used : "<<promo<<"\n";
        cout<<"Paid       : Rs. "<<fixed<<setprecision(2)<<paid<<"\n";
        cout<<"Valid Till : "<<formatDate(s.dueDate)<<"\n";
        cout<<"-----------------------------------------\n";

        ofstream out("receipt_"+to_string(s.userId)+".txt",ios::app);
        out<<"Type: "<<type<<" | ID: "<<s.userId<<" | Plan: "<<s.plan->getTierName()
           <<" | Paid: Rs."<<paid<<" | Valid Till: "<<formatDate(s.dueDate)<<"\n";
        out.close();
    }

    void viewPlans(){
        NormalPlan n;
        SilverPlan s;
        PremiumPlan p;
        cout<<"\n--------------------------------------------------------------------------\n";
        cout<<left<<setw(10)<<"Tier"<<setw(12)<<"1 Month"<<setw(12)<<"3 Months"<<setw(12)<<"1 Year"<<setw(10)<<"Devices"<<"Grace\n";
        cout<<"--------------------------------------------------------------------------\n";
        cout<<left<<setw(10)<<n.getTierName()<<setw(12)<<n.calculatePrice(1)<<setw(12)<<n.calculatePrice(3)<<setw(12)<<n.calculatePrice(12)<<setw(10)<<n.getMaxDevices()<<n.getGraceDays()<<"d\n";
        cout<<left<<setw(10)<<s.getTierName()<<setw(12)<<s.calculatePrice(1)<<setw(12)<<s.calculatePrice(3)<<setw(12)<<s.calculatePrice(12)<<setw(10)<<s.getMaxDevices()<<s.getGraceDays()<<"d\n";
        cout<<left<<setw(10)<<p.getTierName()<<setw(12)<<p.calculatePrice(1)<<setw(12)<<p.calculatePrice(3)<<setw(12)<<p.calculatePrice(12)<<setw(10)<<p.getMaxDevices()<<p.getGraceDays()<<"d\n";
        cout<<"--------------------------------------------------------------------------\n";
        cout<<"Promos: WELCOME10, STUDENT20, ANNUAL25\n";
    }

    void addSubscriber(){
        if(count>=100){
            cout<<"Database full!\n";
            return;
        }
        Subscriber s;
        s.userId=nextId++;
        cin.ignore();
        cout<<"Enter Full Name: ";
        getline(cin,s.name);
        cout<<"Enter Email: ";
        getline(cin,s.email);
        cout<<"Enter Mobile: ";
        getline(cin,s.mobile);

        s.plan=selectPlan();
        s.durationMonths=selectDuration();
        double base=s.plan->calculatePrice(s.durationMonths);
        string promo;
        double finalAmt=applyPromo(base,s.durationMonths,promo);

        s.amountPaid=finalAmt;
        s.startDate=time(0);
        s.dueDate=addMonths(s.startDate,s.durationMonths);
        s.isActive=true;

        list[count]=s;
        count++;
        saveToFile();
        printReceipt(s,finalAmt,promo,"NEW SUBSCRIPTION");
    }

    void showHeader(){
        cout<<"\n-------------------------------------------------------------------------------------------\n";
        cout<<left<<setw(7)<<"ID"<<setw(18)<<"Name"<<setw(22)<<"Email"<<setw(10)<<"Tier"<<setw(7)<<"Plan"<<setw(13)<<"Due Date"<<"Status\n";
        cout<<"-------------------------------------------------------------------------------------------\n";
    }

    void showRow(Subscriber s){
        cout<<left<<setw(7)<<s.userId
            <<setw(18)<<s.name.substr(0,16)
            <<setw(22)<<s.email.substr(0,20)
            <<setw(10)<<s.plan->getTierName()
            <<setw(7)<<(to_string(s.durationMonths)+"M")
            <<setw(13)<<formatDate(s.dueDate)
            <<s.getStatus()<<"\n";
    }

    void viewAll(){
        if(count==0){
            cout<<"\nNo subscribers found.\n";
            return;
        }
        showHeader();
        for(int i=0;i<count;i++){
            showRow(list[i]);
        }
    }

    int findIndex(int id){
        for(int i=0;i<count;i++){
            if(list[i].userId==id) return i;
        }
        return -1;
    }

    void searchSubscriber(){
        int id;
        cout<<"Enter User ID to search: ";
        cin>>id;
        int idx=findIndex(id);
        if(idx==-1){
            cout<<"Subscriber not found.\n";
            return;
        }
        showHeader();
        showRow(list[idx]);
    }

    void viewDueAndExpired(){
        showHeader();
        bool found=false;
        for(int i=0;i<count;i++){
            string st=list[i].getStatus();
            if(st=="Expired" || st=="Grace Period" || st=="Expiring Soon"){
                showRow(list[i]);
                found=true;
            }
        }
        if(!found) cout<<"No due or expired accounts.\n";
    }

    void renewOrUpgrade(){
        int id;
        cout<<"Enter User ID to Renew/Upgrade: ";
        cin>>id;
        int idx=findIndex(id);
        if(idx==-1){
            cout<<"User ID not found.\n";
            return;
        }
        cout<<"Current Plan: "<<list[idx].plan->getTierName()<<" ("<<list[idx].getStatus()<<")\n";
        list[idx].plan=selectPlan();
        list[idx].durationMonths=selectDuration();

        double base=list[idx].plan->calculatePrice(list[idx].durationMonths);
        string promo;
        double finalAmt=applyPromo(base,list[idx].durationMonths,promo);

        list[idx].amountPaid=list[idx].amountPaid+finalAmt;
        time_t baseDate=time(0);
        if(difftime(list[idx].dueDate,time(0))>=0 && list[idx].isActive){
            baseDate=list[idx].dueDate;
        }
        list[idx].startDate=time(0);
        list[idx].dueDate=addMonths(baseDate,list[idx].durationMonths);
        list[idx].isActive=true;

        saveToFile();
        printReceipt(list[idx],finalAmt,promo,"RENEW / UPGRADE");
    }

    void editDetails(){
        int id;
        cout<<"Enter User ID to Edit: ";
        cin>>id;
        int idx=findIndex(id);
        if(idx==-1){
            cout<<"User ID not found.\n";
            return;
        }
        cin.ignore();
        cout<<"Enter New Name: ";
        getline(cin,list[idx].name);
        cout<<"Enter New Email: ";
        getline(cin,list[idx].email);
        cout<<"Enter New Mobile: ";
        getline(cin,list[idx].mobile);
        saveToFile();
        cout<<"Details updated!\n";
    }

    void toggleAccess(){
        int id;
        cout<<"Enter User ID to Revoke/Reactivate: ";
        cin>>id;
        int idx=findIndex(id);
        if(idx==-1){
            cout<<"User ID not found.\n";
            return;
        }
        list[idx].isActive=!list[idx].isActive;
        saveToFile();
        if(list[idx].isActive) cout<<"Access Reactivated.\n";
        else cout<<"Access Revoked.\n";
    }

    void deleteSubscriber(){
        int id;
        cout<<"Enter User ID to Delete: ";
        cin>>id;
        int idx=findIndex(id);
        if(idx==-1){
            cout<<"User ID not found.\n";
            return;
        }
        // Array shift karke record delete kar rahe hain
        for(int i=idx;i<count-1;i++){
            list[i]=list[i+1];
        }
        count--;
        saveToFile();
        cout<<"Record deleted.\n";
    }

    void verifyAccess(bool checkEmail){
        int id;
        cout<<"Enter User ID: ";
        cin>>id;
        int idx=findIndex(id);
        if(idx==-1){
            cout<<"User ID not found.\n";
            return;
        }
        if(checkEmail){
            string em;
            cout<<"Enter Registered Email: ";
            cin>>em;
            if(em!=list[idx].email){
                cout<<"Email mismatch!\n";
                return;
            }
        }
        cout<<"\nUser   : "<<list[idx].name<<"\n";
        cout<<"Status : "<<list[idx].getStatus()<<"\n";
        if(list[idx].hasValidAccess()){
            cout<<"Access : GRANTED\n";
            cout<<"Plan   : "<<list[idx].plan->getTierName()<<" ("<<list[idx].plan->getMaxDevices()<<" Devices)\n";
            cout<<"Perks  : "<<list[idx].plan->getFeatures()<<"\n";
            cout<<"Due On : "<<formatDate(list[idx].dueDate)<<"\n";
        }
        else{
            cout<<"Access : DENIED (Please renew your plan)\n";
        }
    }

    void showAnalytics(){
        int active=0,expired=0,normal=0,silver=0,premium=0;
        double revenue=0;
        for(int i=0;i<count;i++){
            revenue=revenue+list[i].amountPaid;
            string st=list[i].getStatus();
            if(st=="Active" || st=="Expiring Soon" || st=="Grace Period") active++;
            else expired++;

            string t=list[i].plan->getTierName();
            if(t=="Normal") normal++;
            else if(t=="Silver") silver++;
            else premium++;
        }
        cout<<"\n--- ANALYTICS REPORT ---\n";
        cout<<"Total Users   : "<<count<<"\n";
        cout<<"Valid Users   : "<<active<<"\n";
        cout<<"Expired/Rev   : "<<expired<<"\n";
        cout<<"Normal Plans  : "<<normal<<"\n";
        cout<<"Silver Plans  : "<<silver<<"\n";
        cout<<"Premium Plans : "<<premium<<"\n";
        cout<<"Total Revenue : Rs. "<<fixed<<setprecision(2)<<revenue<<"\n";
    }
};

void adminMenu(SubscriptionManager &mgr){
    string u,p;
    cout<<"Admin Username: ";
    cin>>u;
    cout<<"Admin Password: ";
    cin>>p;
    if(u!="admin" || p!="admin123"){
        cout<<"Wrong username or password!\n";
        return;
    }

    int ch;
    while(true){
        cout<<"\n--- ADMIN MENU ---\n";
        cout<<"1. Add Subscriber\n";
        cout<<"2. View All Subscribers\n";
        cout<<"3. Search Subscriber\n";
        cout<<"4. View Expiring/Grace/Expired\n";
        cout<<"5. Renew / Upgrade Plan\n";
        cout<<"6. Edit Subscriber Details\n";
        cout<<"7. Revoke / Reactivate Access\n";
        cout<<"8. Delete Subscriber\n";
        cout<<"9. Verify App Access\n";
        cout<<"10. View Analytics\n";
        cout<<"11. View Plan Catalog\n";
        cout<<"0. Logout\n";
        cout<<"Enter choice: ";
        cin>>ch;

        if(ch==1) mgr.addSubscriber();
        else if(ch==2) mgr.viewAll();
        else if(ch==3) mgr.searchSubscriber();
        else if(ch==4) mgr.viewDueAndExpired();
        else if(ch==5) mgr.renewOrUpgrade();
        else if(ch==6) mgr.editDetails();
        else if(ch==7) mgr.toggleAccess();
        else if(ch==8) mgr.deleteSubscriber();
        else if(ch==9) mgr.verifyAccess(false);
        else if(ch==10) mgr.showAnalytics();
        else if(ch==11) mgr.viewPlans();
        else if(ch==0) break;
        else cout<<"Invalid choice!\n";
    }
}

void userMenu(SubscriptionManager &mgr){
    int ch;
    while(true){
        cout<<"\n--- SUBSCRIBER PORTAL ---\n";
        cout<<"1. Check My Subscription Status\n";
        cout<<"2. View Available Plans\n";
        cout<<"0. Back\n";
        cout<<"Enter choice: ";
        cin>>ch;

        if(ch==1) mgr.verifyAccess(true);
        else if(ch==2) mgr.viewPlans();
        else if(ch==0) break;
        else cout<<"Invalid choice!\n";
    }
}

int main(){
    SubscriptionManager mgr;
    int ch;
    while(true){
        cout<<"\n=== SUBSCRIPTION MANAGEMENT SYSTEM ===\n";
        cout<<"1. Admin Login\n";
        cout<<"2. Subscriber Portal\n";
        cout<<"3. View Plans & Pricing\n";
        cout<<"0. Exit\n";
        cout<<"Enter choice: ";
        cin>>ch;

        if(ch==1) adminMenu(mgr);
        else if(ch==2) userMenu(mgr);
        else if(ch==3) mgr.viewPlans();
        else if(ch==0){
            cout<<"Exiting...\n";
            break;
        }
        else cout<<"Invalid choice!\n";
    }
    return 0;
}
