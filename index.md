# IE360 Project Report
Group 8:

_Süheyla Şeker_

_Doğanay Tuhan _

_Erdem Çağrı Utku_

_Yağmur Karalar_

_Abdulsamed Kağıt_


**Introduction**

Demand plays an important role in the management of every business. Almost every organization face some internal and external risks such as high competition, recession, inflation and high inventory cost. An organization can lessen the negative effects of these risks by forecasting the demand for its products and services in future.

Trendyol is an e-commerce website selling wide range of products. In this project, we are tasked with developing forecasting models to predict the next day’s demand of 8 specific products. The historical data about these products which might be related to selling quantity were provided by Trendyol. The data includes:

* **Price**: Average price of the items sold for the given day.

* **Event Date**: Date of the event. 

* **Sold Count**: Number of items sold in the given day. 

* **Visit Count**: Number of visits to the particular product. 

* **Favored Count**: Number of times the product is added to favorites.

* **Basket Count**: Number of times the product is added to basket (not necessarily purchased). 

* **Category Sold**: Number of items sold from the same category.

* **Category Brand Sold**: Number of items sold from the same category brand.

* **Category Visits**: Visits to the category of the item in the given day.

* **TY Visits**: Total Trendyol visits in the given day.

Apart from these, we thought daily coronavirus cases data might be useful in forecasting since coronavirus outbreak caused a significant rise in online shopping.
Two alternative forecasting models have been developed: ETS and Arima with regression. These models were applied to each product and better alternative was selected to make forecast. Input variables for Arima with regression are explained detaily in Approach part. There were some NA values in given data which needed to be preprocessed. These values are replaced by most recent “non-NA” values after to them.

**Approach**

We used two main models in this project. One of these two models is used for each of the eight products. In our first model, we focused on building an Arima model with regression. The input variables we used in the regression were: **price_difference**, **coronavirusdailycases**, **visit_count**, **basket_count**, **price**, **category_sold**. Price_difference is the difference between consecutive daily prices of the product. The reason why we added this to the regression model is that it is expected to capture the sudden changes in demand at times of considerable discounts and rises in prices. 

Although the price parameter can capture the change in demand due to discounts and price increases in the long term, the initial effects of these are usually stronger than their long term effects, and the price column falls short of capturing the initial surges or reductions in the demand. The spectacular increase in demand in black Fridays is a nice example of these sudden surges in demand. 

The coronavirusdailycases column records the number of new coronavirus cases in Turkey for each day. This column is made 0 for all the days before the first coronavirus case in Turkey. The intuition behind using this column is simple: The possibility of contracting coronavirus causes people to prefer online shopping, and the more number of coronavirus cases is expected to bring more Trendyol sales. The information for this column is updated each day by importing an Excel storing the up-to-date number of coronavirus cases from the following website: https://ourworldindata.org/coronavirus-source-data. The columns visit_count, basket_count, price, and category_sold were already present in the given data and we used them in the model directly.

Not all of the columns that came with the data were used in the model. category_visits and ty_visits are removed because their values were NA in the first couple of months, which caused too many missing values to handle. product_content_id is removed because it is not a value that helps to predict the output. We removed favored_count because it was highly correlated with visit_count, and visit_count was the one that is more correlated with the output (better predictor) among these two. We removed category_brand_sold because it was highly correlated with category _sold, and category_sold was the one that is more correlated with the output (better predictor) among these two. 

There is no need to include predictors that are very much correlated with each other because using highly correlated predictors will unnecessarily increase the complexity of the model and make it less interpretable. When we built an Arima model without regression and fitted it to the product with id 7061886, our performance parameters came out to be: AIC=2864.14, AICc=2864.28, BIC=2878.86. When we built an Arima model with regression using the abovementioned predictors and fitted it to the same product, our performance parameters came out to be: AIC=2211.21, AICc=2211.85, BIC=2244.36. This proves that our predictors significantly improved the model. When we applied the Ljung-Box test on the Arima with regression, the p-value came out to be 0.2311 and the residuals looked like white noise, which is a good sign for our model. 

We also tried fitting the ETS model for comparison. According to the fitted ETS model, our data has neither trend nor seasonality, and it has additive errors. When we applied cross-validation on both the ETS model and the Arima model with regression, we saw that the latter gives a much better result, that’s why we went ahead with it. The only thing missing was forecasting our predictors, and we forecasted all of our predictors using the tbats() function before using them in the model.Arima model with regression performed better than ETS model for all product types.


**Results**

The forecasts created by the aforementioned approach are submitted to the system via project API. Each day, predictors and sales counts are updated and new forecasts created by the models are submitted to the system for each product. What we have seen in our submissions was that, until the end of May, our group ranked in top 5 almost half of the days, with an upwards trend in the last week. However, June rankings were considerably lower possibly because consumer habits in summer is different than the spring season. Another reason could be that, considering most of the products are prevalently used in summer, sales count for these products may have jumped higher than the expectations of our model. The reason for the lower performance may be that, seemingly, people of Turkey got used to the pandemic in June, and the number of corona cases may have become less effective on the sales counts. Overall, our group ranked 6th among the 23 groups, which could be higher if a revision in the model was considered for the summer period.  
 
**Conclusion and Future Work**

There were two main ways of improving our Arima model with regression (the first model): Adding more predictors and improving the forecasts of our predictors. The former can be managed by analyzing the root causes of the fluctuations in the demand so as to add new predictors to account for these fluctuations. The latter can be managed by trying out various models for our predictors so as to find the most accurate model for forecasting them. We confined ourselves to relying on the tbats() function for all of our predictors, but doing more work on this could have given us a better forecasting model for each of our predictors. The worst aspect of using regression in forecasting is the need to forecast the predictors as well, and not forecasting the predictors accurately will result in a poor prediction for the target even if the main model happens to be quite successful.

[Here](files/FullCode.R) is the full R code.

```{r install.packages, include=FALSE}
require(tsbox)
require(jsonlite)
require(httr)
require(data.table)
library(xts)
library(zoo)
library(ggfortify)
library(ggplot2)
library(fpp2)

get_token <- function(username, password, url_site){
  
  post_body = list(username=username,password=password)
  post_url_string = paste0(url_site,'/token/')
  result = POST(post_url_string, body = post_body)
  
  # error handling (wrong credentials)
  if(result$status_code==400){
    print('Check your credentials')
    return(0)
  }
  else if (result$status_code==201){
    output = content(result)
    token = output$key
  }
  
  return(token)
}

get_data <- function(start_date='2020-03-20', token, url_site){
  
  post_body = list(start_date=start_date,username=username,password=password)
  post_url_string = paste0(url_site,'/dataset/')
  
  header = add_headers(c(Authorization=paste('Token',token,sep=' ')))
  result = GET(post_url_string, header, body = post_body)
  output = content(result)
  data = data.table::rbindlist(output)
  data[,event_date:=as.Date(event_date)]
  data = data[order(product_content_id,event_date)]
  return(data)
}


send_submission <- function(predictions, token, url_site, submit_now=T){
  
  format_check=check_format(predictions)
  if(!format_check){
    return(FALSE)
  }
  
  post_string="list("
  for(i in 1:nrow(predictions)){
    post_string=sprintf("%s'%s'=%s",post_string,predictions$product_content_id[i],predictions$forecast[i])
    if(i<nrow(predictions)){
      post_string=sprintf("%s,",post_string)
    } else {
      post_string=sprintf("%s)",post_string)
    }
  }
  
  submission = eval(parse(text=post_string))
  json_body = jsonlite::toJSON(submission, auto_unbox = TRUE)
  submission=list(submission=json_body)
  
  print(submission)
  # {"31515569":2.4,"32939029":2.4,"4066298":2.4,"6676673":2.4,"7061886":2.4,"85004":2.4} 
  
  if(!submit_now){
    print("You did not submit.")
    return(FALSE)      
  }
  
  
  header = add_headers(c(Authorization=paste('Token',token,sep=' ')))
  post_url_string = paste0(url_site,'/submission/')
  result = POST(post_url_string, header, body=submission)
  
  if (result$status_code==201){
    print("Successfully submitted. Below you can see the details of your submission")
  } else {
    print("Could not submit. Please check the error message below, contact the assistant if needed.")
  }
  
  print(content(result))
  
}


check_format <- function(predictions){
  
  if(is.data.frame(predictions) | is.data.frame(predictions)){
    if(all(c('product_content_id','forecast') %in% names(predictions))){
      if(is.numeric(predictions$forecast)){
        print("Format OK")
        return(TRUE)
      } else {
        print("forecast information is not numeric")
        return(FALSE)                
      }
    } else {
      print("Wrong column names. Please provide 'product_content_id' and 'forecast' columns")
      return(FALSE)
    }
    
  } else {
    print("Wrong format. Please provide data.frame or data.table object")
    return(FALSE)
  }
  
}

# this part is main code
subm_url = 'http://167.172.183.67'

u_name = "Group8"
p_word = "D0rlLLrPf3oSfFIl"
submit_now = FALSE

username = u_name
password = p_word

token = get_token(username=u_name, password=p_word, url=subm_url)
data = get_data(token=token,url=subm_url)

data[data==(-1)]<-NA


coronavirusdata<-fread("C:/Users/Abdulsamed/Downloads/new_cases.csv", select =c("date","Turkey"))
coronavirusdata$date=as.POSIXct(coronavirusdata$date)
coronavirusdata<-xts(coronavirusdata$Turkey, order.by=coronavirusdata$date)
dates <- seq(from=as.Date("2019-04-30"),to=as.Date("2019-12-30"), by="days")
zeros<-rep(0,length(dates))
extendeddata<-xts(zeros, order.by=dates)
coronavirusdata<-rbind(coronavirusdata,extendeddata)

x<-nrow(data)/8
coronavirusdata<-coronavirusdata[1:x,]

coronavirusdata[is.na(coronavirusdata)] <- 0

product6<-data[data$product_content_id=='7061886']


product6$price<-na.locf(product6$price, fromLast=TRUE)

price_difference <- diff(product6$price)
price_difference<-c(0,price_difference)
product6<-product6[,price_difference:=price_difference]

product6<-product6[,coronavirusdailycases:= coronavirusdata]
a<-head(which(product6[,"visit_count"]>0),n=1)
product6<-product6[-(1:(a-1)),]
product6<-product6[,c("category_visits","ty_visits", "product_content_id"):=NULL]
#category_visits and ty_visits are removed because they have their values are NA in the first couple of months. product_content_id is removed because it is not a value that helps predicting the output.
product6<-na.locf(product6)

product6<-na.locf(product6, fromLast=TRUE)
library(leaps)
cor(product6[,-"event_date"])
#favored_count and visit_count are highly correlated, there is no need to include both of them as predictors. We will remove favored_count because it is less correlated with sold_count.
cor(product6[,-c("event_date","basket_count")])

#category_brand_sold and category_sold are highly correlated, there is no need to include both of them as predictors. We will remove category_brand_sold because it is less correlated with sold_count.
cor(product6[,-c("event_date","basket_count","category_brand_sold")])

#Correlations seem fine now.

product6xts<-xts(product6[,-2], order.by=product6$event_date)
trainxreg<-product6xts[,c("price_difference", "coronavirusdailycases", "visit_count", "basket_count", "price", "category_sold")]

arimafit<-auto.arima(product6xts[,"sold_count"],xreg=trainxreg)
summary(arimafit)
# Regression with ARIMA(1,0,0).  

summary(auto.arima(product6xts[,"sold_count"]))

#Using regression significantly improves our model, we have lower AIC, AICc, and BIC when the input variables are used in the Arima model.

checkresiduals(arimafit)  

# p-value of the Ljung-Box test shows that residuals look like white noise.

summary(ets(product6xts[,"sold_count"]))

# ETS(A,N,N) Our data has neither trend nor seasonality, and it has additive errors.

farima <- function(x, h, xreg) {
  forecast(auto.arima(x, xreg = xreg), h=h)}
fets <- function(x, h) {
  forecast(ets(x), h = h)}
e1 <- tsCV(product6xts[,"sold_count"], farima, h=1, xreg=trainxreg)

e2 <- tsCV(product6xts[,"sold_count"], fets, h=1)
mean(e1^2, na.rm=TRUE)

mean(e2^2, na.rm=TRUE)

#By comparing errors, we see that the Arima model gives better results, that’s why we will go ahead with it.


library(tsbox)
product6ts <-ts_ts(product6xts)
product6ts<-ts(product6ts, frequency=7) 
A<-(forecast(tbats(product6ts[,"price_difference"]),h=1))$mean[1]
B<-(forecast(tbats(product6ts[,"coronavirusdailycases"]),h=1))$mean[1]
C<-(forecast(tbats(product6ts[,"visit_count"]),h=1))$mean[1]
D<-(forecast(tbats(product6ts[,"basket_count"]),h=1))$mean[1]
E<-(forecast(tbats(product6ts[,"price"]),h=1))$mean[1]
G<-(forecast(tbats(product6ts[,"category_sold"]),h=1))$mean[1]

product6prediction <- round(forecast(arimafit, xreg=cbind(A,B,C,D,E,G))$mean[1], digit=0)
product6prediction
predictions=unique(data[,list(product_content_id)])
predictions[7,forecast:=product6prediction]

```
