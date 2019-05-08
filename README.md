# Sentiment
Sentiment analysis
package au.edu.curtin.solrDataExtract9;

import java.sql.*;
import java.io.FileReader;
import java.io.IOException;
import java.util.Properties;
import org.json.simple.JSONObject;
import org.json.simple.JSONArray;
import org.json.simple.parser.JSONParser;
import org.json.simple.parser.ParseException;
import edu.stanford.nlp.ling.CoreAnnotations;
import edu.stanford.nlp.neural.rnn.RNNCoreAnnotations;
import edu.stanford.nlp.pipeline.Annotation;
import edu.stanford.nlp.pipeline.StanfordCoreNLP;
import edu.stanford.nlp.sentiment.SentimentCoreAnnotations;
import edu.stanford.nlp.sentiment.SentimentCoreAnnotations.SentimentAnnotatedTree;
import edu.stanford.nlp.trees.Tree;
import edu.stanford.nlp.util.CoreMap;

public class codeForDissertation{
	//Create a pipeline of the Stanford CoreNLP
		static StanfordCoreNLP pipeline = null;
		//Create a string variable for recording sentiment result
		static String scoreStr;
		//Initiate the Stanford pipeline
		//Reference: https://stanfordnlp.github.io/CoreNLP/api.html
	    public static void init() 
	    {
	    	//Set up pipeline properties
	        Properties props = new Properties();
	        //Set the list of annotators to run
	        props.setProperty("annotators", "tokenize, ssplit, parse, sentiment");
	        //Build pipeline
	        pipeline = new StanfordCoreNLP(props);
	    }
    
	public static void main(String[] args) throws IOException, ParseException{  
		init();
    	try{  
    		//Connect mysql database server for data storing.
    		//Reference: https://www.javatpoint.com/example-to-connect-to-the-mysql-database
    		Class.forName("com.mysql.jdbc.Driver");  
    		Connection con=DriverManager.getConnection(
    		"jdbc:mysql://localhost:3306/sonoo","root","root");  
    		con.createStatement();  
    		
    		//Read JSON file
	        JSONParser parser = new JSONParser();
	        JSONArray array = (JSONArray) parser.parse(new FileReader("...\\20190228_2.json"));
	        //Create a for loop to go through each items of JSON file.
	        for(int i = 0; i < array.size(); i++) {
			JSONObject jsonObject = (JSONObject) array.get(i);
			
			String content = (String) jsonObject.get("content").toString().trim();
        	String id = (String) jsonObject.get("id");
            String title = (String) jsonObject.get("title");            
            String tstamp = (String) jsonObject.get("tstamp");

            //Call the method to clean content, title, tstamp.
            content = contentClean(content);         	            
            title = dataContent(title);
            tstamp = dataContent(tstamp);
			content = dataContent(content);
    		
			//Call the method to seperate content into single sentence.
			String[] sentenceOfContent = getSentence(content);
			
			int numberSentence = 1;
			int articleID = i+1;
			
			//Create a for loop to go through each sentence to get sentiment score and output into database.
    		for (int k=0 ; k < sentenceOfContent.length; k++)
    		{
    			String singleSentence = sentenceOfContent[k].toString();
        		
    			//Call the method to obtain sentiment score.
        		int score = sentiment(singleSentence);
        			
        		//The mysql insert statement
        	    String query = " insert into sentiment (article_id, url, title, tstamp, sentence_id, sentence, sentiment_score, sentiment)"
        	          + " values (?, ?, ?, ?, ?, ?, ?, ?)";

        	    //Create the mysql insert preparedstatement to output each elements.
        	    PreparedStatement preparedStmt = con.prepareStatement(query);
        	    preparedStmt.setInt (1, articleID);
        	    preparedStmt.setString (2, id);
        	    preparedStmt.setString (3, title);
        	    preparedStmt.setString (4, tstamp);
        	    preparedStmt.setInt (5, numberSentence);
        	    preparedStmt.setString (6, singleSentence);
        	    preparedStmt.setInt (7, score);
        	    preparedStmt.setString (8, scoreStr);
        	    //Execute the preparedstatement
        	    preparedStmt.execute();
        	        
        	    numberSentence++;
        	    System.out.println("sucess" + articleID);
        	}
    	}	
		con.close(); 
		}catch(Exception e){ System.out.println(e);}  
	}

	//Data clean method
	//Reference: https://howtodoinjava.com/regex/java-clean-ascii-text-non-printable-chars/
    public static String dataContent(String text) throws IOException 
    {
        //Strips off all non-ASCII characters
        text = text.replaceAll("[^\\x00-\\x7F]", "");
        //Erases all the ASCII control characters
        text = text.replaceAll("[\\p{Cntrl}&&[^\r\n\t]]", "");
        //Removes non-printable characters from Unicode
        text = text.replaceAll("\\p{C}", "");
        text = text.replaceAll("[â€™]", "'");
        text = text.replaceAll("[â€˜]", "'");
        text = text.replaceAll("http.*?\\s", "");
        text = text.replaceAll("&", "");
        text = text.replaceAll("-", "");
        text = text.replaceAll("@", "");
		return text;
    } 
    
    //Content filter method
    //This method aims to exclude the content that has less than 200 words and has less than average 5 sentence.
    //These content that is described above are name of advertisement.
    public static String contentClean(String content) {	
        String []wordsOfContent = content.split("\\W+");
        String []lineOfContent = content.split("[\\r?\\n\\|\\.]+");
        int AverageSentenceOfContent = wordsOfContent.length/lineOfContent.length;
        if(wordsOfContent.length <200 || AverageSentenceOfContent <5) {
        		content = "";
  		}
		return content;
	}

    //Separate content into sentence method 
    private static String[] getSentence(String content) throws IOException {
    	String[]sentenceOfContent = content.split("[\\r?\\n\\|\\.]+");
	    return sentenceOfContent;	
    }
    
    //Calculate sentiment score method
    //Reference: https://stanfordnlp.github.io/CoreNLP/api.html
    public static int sentiment(String singleSentence) throws IOException
    {
 		int score = 2; //Default as Neutral. 1 = Negative, 2 = Neutral, 3 = Positive
		Annotation annotation = pipeline.process(singleSentence);
    	for(CoreMap sentence : annotation.get(CoreAnnotations.SentencesAnnotation.class))
    	{
    		scoreStr = sentence.get(SentimentCoreAnnotations.SentimentClass.class);
    		Tree tree = sentence.get(SentimentAnnotatedTree.class);
    		score = RNNCoreAnnotations.getPredictedClass(tree);
    		System.out.println(scoreStr + "\t" + score + "\t" + sentence);
    	}
		return score;
    }
}
