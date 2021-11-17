public class ProjectCalloutService {
      @InvocableMethod
	public static void postOpportunityToPMS(List<Id> opptyId){
        Opportunity oppty = [SELECT Id,Account.Name
,Name,CloseDate,Amount FROM Opportunity WHERE Id =: opptyId[0]];
        String sToken = ServiceTokens__c.getValues('ProjectServiceToken').Token__c;
        
        OpptyWrapper opptyWrapper = new OpptyWrapper(oppty.Id
,oppty.Name
,oppty.Account.Name
,oppty.CloseDate,(Integer)oppty.Amount);
        String jsonInput = json.serialize(opptyWrapper);
        
        
        System.enqueueJob(new QueueablePMSCall(sToken,jsonInput,oppty.Id));
    }
    
    public class OpptyWrapper {
        public String opportunityId {get; set;}
        public String opportunityName {get; set;}
        public String accountName {get; set;}
        public Date closeDate {get; set;}
        public Integer amount {get; set;}
        public OpptyWrapper(String opportunityId, String opportunityName, String accountName, Date closeDate, Integer amount){
            this.opportunityId = opportunityId;
            this.opportunityName = opportunityName;
            this.accountName = accountName;
            this.closeDate = closeDate;
			this.amount = amount;
        }            
        
    }
    
    
    
    class QueueablePMSCall implements System.Queueable,Database.AllowsCallouts{
        private String sToken;
        private String jsonInput;
        private Id opptyId;
        
        public QueueablePMSCall(String sToken, string jsonInput, Id opptyId){
           	this.sToken = sToken;
            this.jsonInput = jsonInput;
            this.opptyId = opptyId;
        }
        
        public void execute(QueueableContext qc){
            postToPMS(sToken,jsonInput,opptyId);
        }
    }
    
    @Future(callout=true)
        private static void postToPMS(String sToken,String jsonInput,Id opptyId){
            HTTPRequest req = new HTTPRequest();
            req.setEndpoint('callout:ProjectService');
            req.setMethod('POST');
            req.setHeader('token', sToken);
            req.setHeader('Content-Type','application/json;charset=UTF-8');
            req.setBody(jsonInput);
            
            HTTP http = new HTTP();
            HTTPResponse res = http.send(req);
            
            Opportunity oppty = new Opportunity(Id=opptyId);
            if (res.getStatusCode() == 201){
                
            } else {
                oppty.StageName = 'Resubmit Project';
            }
           update oppty;
        }
}
