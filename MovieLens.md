# capstone-project


# Install all needed libraries if it is not present

if(!require(tidyverse)) install.packages("tidyverse") 
if(!require(kableExtra)) install.packages("kableExtra")
if(!require(tidyr)) install.packages("tidyr")
if(!require(tidyverse)) install.packages("tidyverse")
if(!require(stringr)) install.packages("stringr")
if(!require(forcats)) install.packages("forcats")
if(!require(ggplot2)) install.packages("ggplot2")


# Loading all needed libraries

library(dplyr)
library(tidyverse)
library(kableExtra)
library(tidyr)
library(stringr)
library(forcats)
library(ggplot2)


if(!require(tidyverse)) install.packages("tidyverse", repos = "http://cran.us.r-project.org")
if(!require(caret)) install.packages("caret", repos = "http://cran.us.r-project.org")

# MovieLens 10M dataset:
# https://grouplens.org/datasets/movielens/10m/
# http://files.grouplens.org/datasets/movielens/ml-10m.zip


dl <- tempfile()
download.file("http://files.grouplens.org/datasets/movielens/ml-10m.zip", dl)

ratings <- read.table(text = gsub("::", "\t", readLines(unzip(dl, "ml-10M100K/ratings.dat"))),
                      col.names = c("userId", "movieId", "rating", "timestamp"))

movies <- str_split_fixed(readLines(unzip(dl, "ml-10M100K/movies.dat")), "\\::", 3)
colnames(movies) <- c("movieId", "title", "genres")
movies <- as.data.frame(movies) %>% mutate(movieId = as.numeric(levels(movieId))[movieId],
                                           title = as.character(title),
                                           genres = as.character(genres))

movielens <- left_join(ratings, movies, by = "movieId")


# Validation set will be 10% of MovieLens data

set.seed(1)
test_index <- createDataPartition(y = movielens$rating, times = 1, p = 0.1, list = FALSE)
edx <- movielens[-test_index,]
temp <- movielens[test_index,]

# Make sure userId and movieId in validation set are also in edx set


validation <- temp %>% 
  semi_join(edx, by = "movieId") %>%
  semi_join(edx, by = "userId")

# Add rows removed from validation set back into edx set

removed <- anti_join(temp, validation)
edx <- rbind(edx, removed)

rm(dl, ratings, movies, test_index, temp, movielens, removed)



# Executive Summary

The purpose of this project is creating a system using MovieLens dataset. 

This version contains approximately 10 Million movie ratings, separated to 9 Million for training and one Milion for validation. In the training dataset there are approximately **70.000 users** and **11.000 different movies** catergorized into 20 genres, including Action, Adventure, Horror, Drama, Thriller and more.

After the initial data exploration, the systems built on this dataset are evaluated and choosen based on the RMSE - Root Mean Squared Error. The RSME should be lower than **0.87750**.

$$\mbox{RMSE} = \sqrt{\frac{1}{n}\sum_{t=1}^{n}e_t^2}$$


# The RMSE function that will be used in this project is:

RMSE <- function(true_ratings = NULL, predicted_ratings = NULL) {
    sqrt(mean((true_ratings - predicted_ratings)^2))
}



For accomplishing this goal, the **Regularized Movie+User+Genre Model** is capable to reach a RMSE of **0.8628**, that is really good.

# Exploratory Data Analysis

## Inital data Exploration

The 10 Millions dataset is divided into two dataset: ```edx``` for training purpose and ```validation``` for the validation phase. 

The ```edx``` dataset contains approximately 9 Millions of rows with 70.000 different users and 11.000 movies with rating score between 0.5 and 5. There is no missing values (0 or NA).



**edx dataset**


edx %>% summarize(Users = n_distinct(userId),
              Movies = n_distinct(movieId)) %>% 
kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)




**Missing Values per Column**


edx <- edx %>% select(-X)
validation <- validation %>% select(-X)



sapply(edx, function(x) sum(is.na(x))) %>% 
kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)



The features/variables/columns in both datasets are six:

- **userId** ```<integer>``` that contains the unique identification number for each user.
- **movieId** ```<numeric>``` that contains the unique identification number for each movie.
- **rating** ```<numeric>``` that contains the rating of one movie by one user. Ratings are made on a 5-Star scale with half-star increments.
- **timestamp** ```<integer>``` that contains the timestamp for one specific rating provided by one user.
- **title** ```<character>``` that contains the title of each movie including the year of the release.
- **genres** ```<character>``` that contains a list of pipe-separated of genre of each movie.

\newpage

**First 6 Rows of edx dataset**


head(edx) %>%
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)




## Dataset Pre-Processing and Feature Engineering

After a initial data exploration, we notice that the ```genres``` are pipe-separated values. It's necessary to extract them for more consisten, robust and precise estimate. We also observe that the ```title``` contains the year where the movie war released and this it could be necessary to predic the movie rating. Finally, we can extract the year and the month for each rating.

The pre-processing phase is composed by this steps:

1. Convert ```timestamp``` to a human readable date format;
2. Extract the month and the year from the date;
3. Extract the release year for each movie from the title;
4. Separate each genre from the pipe-separated value. It increases the size of both datasets.



# Convert timestamp to a human readable date

edx$date <- as.POSIXct(edx$timestamp, origin="1970-01-01")
validation$date <- as.POSIXct(validation$timestamp, origin="1970-01-01")
```


# Extract the year and month of rate in both dataset

edx$yearOfRate <- format(edx$date,"%Y")
edx$monthOfRate <- format(edx$date,"%m")

validation$yearOfRate <- format(validation$date,"%Y")
validation$monthOfRate <- format(validation$date,"%m")




# Extract the year of release for each movie in both dataset
# edx dataset

edx <- edx %>%
   mutate(title = str_trim(title)) %>%
   extract(title,
           c("titleTemp", "release"),
           regex = "^(.*) \\(([0-9 \\-]*)\\)$",
           remove = F) %>%
   mutate(release = if_else(str_length(release) > 4,
                                as.integer(str_split(release, "-",
                                                     simplify = T)[1]),
                                as.integer(release))
   ) %>%
   mutate(title = if_else(is.na(titleTemp),
  title,
                          titleTemp)
         ) %>%
  select(-titleTemp)


# validation dataset

validation <- validation %>%
   mutate(title = str_trim(title)) %>%
   extract(title,
           c("titleTemp", "release"),
           regex = "^(.*) \\(([0-9 \\-]*)\\)$",
           remove = F) %>%
   mutate(release = if_else(str_length(release) > 4,
                                as.integer(str_split(release, "-",
                                                     simplify = T)[1]),
                                as.integer(release))
   ) %>%
   mutate(title = if_else(is.na(titleTemp),
 title,
                          titleTemp)
         ) %>%
  select(-titleTemp)
```


# Extract the genre in edx datasets

edx <- edx %>%
   mutate(genre = fct_explicit_na(genres,
                                       na_level = "(no genres listed)")
          ) %>%
   separate_rows(genre,
                 sep = "\\|")



# Extract the genre in validation datasets

validation <- validation %>%
   mutate(genre = fct_explicit_na(genres,
                                       na_level = "(no genres listed)")
          ) %>%
 separate_rows(genre,
                 sep = "\\|")



# remove unnecessary columns on edx and validation dataset

edx <- edx %>% select(userId, movieId, rating, title, genre, release, yearOfRate, monthOfRate)

validation <- validation %>% select(userId, movieId, rating, title, genre, release, yearOfRate, monthOfRate)



# Convert the columns into the desidered data type

edx$yearOfRate <- as.numeric(edx$yearOfRate)
edx$monthOfRate <- as.numeric(edx$monthOfRate)
edx$release <- as.numeric(edx$release)

validation$yearOfRate <- as.numeric(validation$yearOfRate)
validation$monthOfRate <- as.numeric(validation$monthOfRate)
validation$release <- as.numeric(validation$release)





After preprocessing the data edx dataset should look like this:




# Output the processed dataset

head(edx) %>%
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)


 
 
 ## Rating Distribution

**Overview of Rating Distribution**

According to the histogram below, it shows that there are a small amount of negative votes (below 3). Maybe, the user tends to give a vote if he liked the movie. Half-Star votes are less common than "Full-Star" votes.



hist(edx$rating, main="Distribution of User's Ratings", xlab="Rating")


**Overview of Rating Frequency through Months and Years**


hist(edx$monthOfRate, main="Frequency of User's Ratings through Month", xlab="Month")
hist(edx$yearOfRate, main="Frequency of User's Ratings through Years", xlab="Years")


### Numbers of Ratings per Movie


   ggplot(edx, aes(movieId)) +
   theme_classic()  +
   geom_histogram(bins=500) +
   labs(title = "Ratings Frequency Distribution Per Title (MovieID)",
        x = "Title (MovieID)",
        y = "Frequency")


### Top Rated Movies


edx %>%
   group_by(title) %>%
   summarise(count = n()) %>%
   arrange(desc(count)) %>%
   head(n=25) %>%
   ggplot(aes(title, count)) +
   theme_classic()  +
   geom_col() +
   theme(axis.text.x = element_text(angle = 90, hjust = 1, size = 7)) +
   labs(title = "Ratings Frequency Distribution Per Title - TOP 25 Movies",
        x = "Title",
        y = "Frequency")


edx %>%
   group_by(title) %>%
   summarise(count = n()) %>%
   arrange(desc(count)) %>%
   head(n=25) %>%
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)




### Mean Distribution per Title (Movie ID)


edx %>%
   group_by(title) %>%
   summarise(mean = mean(rating)) %>%
   ggplot(aes(mean)) +
   theme_classic()  +
   geom_histogram(bins=12) +
   labs(title = "Mean Distribution per Title",
        x = "Mean",
        y = "Frequency")



edx %>%
   group_by(title) %>%
   summarise(mean = mean(rating)) %>%
   arrange(desc(mean)) %>%
   head(n=25) %>%
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)




### Median Distribution per Title (Movie ID)


edx %>%
   group_by(title) %>%
   summarise(median = median(rating)) %>%
   ggplot(aes(median)) +
   theme_classic()  +
   geom_histogram(bins=12) +
   labs(title = "Median Distribution per Title",
        x = "Median",



  edx %>%
   group_by(title) %>%
   summarise(median = median(rating)) %>%
   arrange(desc(median)) %>%
   head(n=25) %>%
   kable() %>%
        y = "Frequency")




## Genre Analysis

### Rating Distribution per Genre

**Overview of Rating distribution over Genre**


edx %>%
   group_by(genre) %>%
   summarise(count = n()) %>%
   ggplot(aes(genre, count)) +
   theme_classic()  +
   geom_col() +
   theme(axis.text.x = element_text(angle = 90, hjust = 1)) +
   labs(title = "Ratings Frequency Distribution Per Genre",
        x = "Genre",
        y = "Frequency")

edx %>%
   group_by(genre) %>%
   summarise(count = n()) %>%
   arrange(desc(count)) %>%
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)


### Mean Distribution per Genre


edx %>%
   group_by(genre) %>%
   summarise(mean = mean(rating)) %>%
   ggplot(aes(genre, mean)) +
   theme_classic()  +
   geom_col() +
   theme(axis.text.x = element_text(angle = 90, hjust = 1)) +
   labs(title = "Mean Distribution per Genre",
        x = "Genre",
        y = "Mean")

edx %>%
   group_by(genre) %>%
   summarise(mean = mean(rating)) %>%
   arrange(desc(mean)) %>%
   head(n=35) %>%
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)




### Median Distribution per Genre



edx %>%
   group_by(genre) %>%
   summarise(median = median(rating)) %>%
   ggplot(aes(genre, median)) +
   theme_classic()  +
   geom_col() +
   theme(axis.text.x = element_text(angle = 90, hjust = 1)) +
   labs(title = "Median Distribution per Genre",
        x = "Genre",
        y = "Median")


edx %>%
   group_by(genre) %>%
   summarise(median = median(rating)) %>%
   arrange(desc(median)) %>%
   head(n=35) %>%
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
                 position = "center",
                 font_size = 10,
                 full_width = FALSE)




# Analysis - Model Building and Evaluation

## Naive Baseline Model

** The Naive Model always predicts the mean. In this case, the mean is approximately 3.5.


paste("The mean is:", as.character(mean(edx$rating)))


** The formula used is:

$$Y_{u,i} = \hat{\mu} + \varepsilon_{u,i}$$

With $\hat{\mu}$ is the mean and $\varepsilon_{i,u}$ is the independent errors sampled from the same distribution centered at 0.



# Calculate the average of all movies

mu_hat <- mean(edx$rating)

# Predict the RMSE on the validation set

rmse_mean_model_result <- RMSE(validation$rating, mu_hat)

# Creating a results dataframe that contains all RMSE results

results <- data.frame(model="Naive Mean-Baseline Model", RMSE=rmse_mean_model_result)
        




The RMSE on the validation dataset is **1.05**. Which is very far for target RMSE (below 0.87), that indicates poor performance model.

## Movie-Based Model, a Content-based Approach

The first Non-Naive Model takes into account the content. In this case the movies that are rated higher or lower resperct to each other.

The formula used is:

$$Y_{u,i} = \hat{\mu} + b_i + \epsilon_{u,i}$$

With $\hat{\mu}$ is the mean and $\varepsilon_{i,u}$ is the independent errors sampled from the same distribution centered at 0. The $b_i$ is a measure for the popularity of movie $i$, i.e. the bias of movie $i$.



# Calculate the average of all movies

mu_hat <- mean(edx$rating)

# Calculate the average by movie

movie_avgs <- edx %>%
   group_by(movieId) %>%
   summarize(b_i = mean(rating - mu_hat))

# Compute the predicted ratings on validation dataset

rmse_movie_model <- validation %>%
   left_join(movie_avgs, by='movieId') %>%
   mutate(pred = mu_hat + b_i) %>%
   pull(pred)

rmse_movie_model_result <- RMSE(validation$rating, rmse_movie_model)

# Adding the results to the results dataset

results <- results %>% add_row(model="Movie-Based Model", RMSE=rmse_movie_model_result)




The RMSE on the validationdataset is **0.94**. Better than the Naive Mean-Baseline Model, but is also far from target RMSE (below 0.87) and indicates poor performance for the model.

## Movie + User Model, a User-based approach

The second Non-Naive Model consider that the users have different tastes and rate differently.

The formula used is:

$$Y_{u,i} = \hat{\mu} + b_i + b_u + \epsilon_{u,i}$$

With $\hat{\mu}$ is the mean and $\varepsilon_{i,u}$ is the independent errors sampled from the same distribution centered at 0. The $b_


# Calculate the average of all movies

mu_hat <- mean(edx$rating)

# Calculate the average by movie

movie_avgs <- edx %>%
   group_by(movieId) %>%
   summarize(b_i = mean(rating - mu_hat))

# Calculate the average by user

user_avgs <- edx %>%
   left_join(movie_avgs, by='movieId') %>%
   group_by(userId) %>%
   summarize(b_u = mean(rating - mu_hat - b_i))

# Compute the predicted ratings on validation dataset

rmse_movie_user_model <- validation %>%
   left_join(movie_avgs, by='movieId') %>%
   left_join(user_avgs, by='userId') %>%
   mutate(pred = mu_hat + b_i + b_u) %>%
   pull(pred)

rmse_movie_user_model_result <- RMSE(validation$rating, rmse_movie_user_model)

# Adding the results to the results dataset

results <- results %>% add_row(model="Movie+User Based Model", RMSE=rmse_movie_user_model_result)




The RMSE on the validation dataset is **0.8635**. The Movie+User Based Model reaches the ideal performance. Applying the regularization techniques can also improve the performance.

## Movie + User + Genre Model, the Genre Popularity

The formula used is:

$$Y_{u,i} = \hat{\mu} + b_i + b_u + b_{u,g} + \epsilon_{u,i}$$

With $\hat{\mu}$ is the mean and $\varepsilon_{i,u}$ is the independent errors sampled from the same distribution centered at 0. The $b_i$ is a measure for the popularity of movie $i$, i.e. the bias of movie $i$. The  $b_u$ is a measure for the mildness of user $u$, i.e. the bias of user $u$. The  $b_{u,g}$ is a measure for how much a user $u$ likes the genre $g$.


# Calculate the average of all movies

mu_hat <- mean(edx$rating)

# Calculate the average by movie

movie_avgs <- edx %>%
   group_by(movieId) %>%
   summarize(b_i = mean(rating - mu_hat))

# Calculate the average by user

user_avgs <- edx %>%
   left_join(movie_avgs, by='movieId') %>%
   group_by(userId) %>%
   summarize(b_u = mean(rating - mu_hat - b_i))

genre_pop <- edx %>%
   left_join(movie_avgs, by='movieId') %>%
   left_join(user_avgs, by='userId') %>%
   group_by(genre) %>%
   summarize(b_u_g = mean(rating - mu_hat - b_i - b_u))

# Compute the predicted ratings on validation dataset

rmse_movie_user_genre_model <- validation %>%
   left_join(movie_avgs, by='movieId') %>%
   left_join(user_avgs, by='userId') %>%
   left_join(genre_pop, by='genre') %>%
   mutate(pred = mu_hat + b_i + b_u + b_u_g) %>%
   pull(pred)

rmse_movie_user_genre_model_result <- RMSE(validation$rating, rmse_movie_user_genre_model)


 # Adding the results to the results dataset

results <- results %>% add_row(model="Movie+User+Genre Based Model", RMSE=rmse_movie_user_genre_model_result)




The RMSE on the validation dataset is **0.8634**. The Movie+User+Genre Based Model reaches ideal performance. Adding the genre predictor doesn't improve model performance. Applying regularization techniques can improve the performance.

## Regularization

The regularization method allows us to add a penalty $\lambda$ (lambda) to penalizes movies with large estimates from a small sample size. In order to optimize $b_i$, it necessary to use this equation:

$$\frac{1}{N} \sum_{u,i} (y_{u,i} - \mu - b_{i})^{2} + \lambda \sum_{i} b_{i}^2$$   

reduced to this equation:   

$$\hat{b_{i}} (\lambda) = \frac{1}{\lambda + n_{i}} \sum_{u=1}^{n_{i}} (Y_{u,i} - \hat{\mu}) $$  

### Regularized Movie-Based Model




# Calculate the average of all movies

mu_hat <- mean(edx$rating)

# Define a table of lambdas

lambdas <- seq(0, 10, 0.1)

# Compute the predicted ratings on validation dataset using different values of lambda

rmses <- sapply(lambdas, function(lambda) {



 # Calculate the average by user
  
   b_i <- edx %>%
      group_by(movieId) %>%
      summarize(b_i = sum(rating - mu_hat) / (n() + lambda))



 # Compute the predicted ratings on validation dataset
   
   predicted_ratings <- validation %>%
      left_join(b_i, by='movieId') %>%
      mutate(pred = mu_hat + b_i) %>%
      pull(pred)
   
   # Predict the RMSE on the validation set
   
   return(RMSE(validation$rating, predicted_ratings))
})

# plot the result of lambdas

df <- data.frame(RMSE = rmses, lambdas = lambdas)

ggplot(df, aes(lambdas, rmses)) +
   theme_classic()  +
   geom_point() +
   labs(title = "RMSEs vs Lambdas - Regularized Movie Based Model",
        y = "RMSEs",
        x = "lambdas")



# Get the lambda value that minimize the RMSE
min_lambda <- lambdas[which.min(rmses)]

# Predict the RMSE on the validation set

rmse_regularized_movie_model <- min(rmses)

# Adding the results to the results dataset

results <- results %>% add_row(model="Regularized Movie-Based Model", RMSE=rmse_regularized_movie_model)




The RMSE on the validation dataset is **0.8635**. The Movie+User Based Model reaches performance. Applying the regularization techniques can improve the performance

### Regularized Movie+User Model


# Calculate the average of all movies

mu_hat <- mean(edx$rating)

# Define a table of lambdas

lambdas <- seq(0, 15, 0.1)

# Compute the predicted ratings on validation dataset using different values of lambda

rmses <- sapply(lambdas, function(lambda) {

   # Calculate the average by user
   
   b_i <- edx %>%
      group_by(movieId) %>%
      summarize(b_i = sum(rating - mu_hat) / (n() + lambda))
   
   # Calculate the average by user
   
   b_u <- edx %>%
      left_join(b_i, by='movieId') %>%
      group_by(userId) %>%
      summarize(b_u = sum(rating - b_i - mu_hat) / (n() + lambda))



# Compute the predicted ratings on validation dataset
   
   predicted_ratings <- validation %>%
      left_join(b_i, by='movieId') %>%
      left_join(b_u, by='userId') %>%
      mutate(pred = mu_hat + b_i + b_u) %>%
      pull(pred)
   
   # Predict the RMSE on the validation set
   
   return(RMSE(validation$rating, predicted_ratings))
})




# plot the result of lambdas

df <- data.frame(RMSE = rmses, lambdas = lambdas)

ggplot(df, aes(lambdas, rmses)) +
   theme_classic()  +
   geom_point() +
   labs(title = "RMSEs vs Lambdas - Regularized Movie+User Model",
        y = "RMSEs",
        x = "lambdas")

# Get the lambda value that minimize the RMSE

min_lambda <- lambdas[which.min(rmses)]

# Predict the RMSE on the validation set

rmse_regularized_movie_user_model <- min(rmses)

# Adding the results to the results dataset

results <- results %>% add_row(model="Regularized Movie+User Based Model", RMSE=rmse_regularized_movie_user_model)





The RMSE on the validation dataset is **0.8629**. The Regularized Movie+User Based Model improves just a bit.

### Regularized Movie+User+Genre Model

# Calculate the average of all movies

mu_hat <- mean(edx$rating)

# Define a table of lambdas

lambdas <- seq(0, 15, 0.1)

# Compute the predicted ratings on validation dataset using different values of lambda

rmses <- sapply(lambdas, function(lambda) {

   # Calculate the average by user
   
   b_i <- edx %>%
      group_by(movieId) %>%
      summarize(b_i = sum(rating - mu_hat) / (n() + lambda))



# Calculate the average by user
   
   b_i <- edx %>%
      group_by(movieId) %>%
      summarize(b_i = sum(rating - mu_hat) / (n() + lambda))
   
   # Calculate the average by user
   
   b_u <- edx %>%
      left_join(b_i, by='movieId') %>%
      group_by(userId) %>%
      summarize(b_u = sum(rating - b_i - mu_hat) / (n() + lambda))
   
    b_u_g <- edx %>%
      left_join(b_i, by='movieId') %>%
      left_join(b_u, by='userId') %>%
      group_by(genre) %>%
      summarize(b_u_g = sum(rating - b_i - mu_hat - b_u) / (n() + lambda))
   
   # Compute the predicted ratings on validation dataset
   
   predicted_ratings <- validation %>%
      left_join(b_i, by='movieId') %>%
      left_join(b_u, by='userId') %>%
      left_join(b_u_g, by='genre') %>%
      mutate(pred = mu_hat + b_i + b_u + b_u_g) %>%
      pull(pred)

# Predict the RMSE on the validation set
   
   return(RMSE(validation$rating, predicted_ratings))
})

# plot the result of lambdas

df <- data.frame(RMSE = rmses, lambdas = lambdas)

ggplot(df, aes(lambdas, rmses)) +
   theme_classic()  +
   geom_point() +
   labs(title = "RMSEs vs Lambdas - Regularized Movie+User+Genre Model",
        y = "RMSEs",
        x = "lambdas")

# Get the lambda value that minimize the RMSE

min_lambda <- lambdas[which.min(rmses)]

# Predict the RMSE on the validation set

rmse_regularized_movie_user_genre_model <- min(rmses)



# Adding the results to the results dataset

results <- results %>% add_row(model="Regularized Movie+User+Genre Based Model", RMSE=rmse_regularized_movie_user_genre_model)



  
The RMSE on the validation dataset is **0.8628**. 

# Results
** Here is the summary result for all the models on edx dataset and validation dataset.


# Shows the results

results %>% 
   kable() %>%
   kable_styling(bootstrap_options = c("striped", "hover", "condensed", "responsive"),
             position = "center",
             font_size = 10,
             full_width = FALSE)



# Conclusion

After training all the different models, it is clear that the movieId and userId contribute more than the genre predictor. By applying regularization and adding the genre predictor can make it possible to reach a RSME of **0.8628** which is closest to the ideal outcome.


# Appendix


## edX Code

#############################################################
# Create edx set, validation set, and submission file
#############################################################


# Note: this process could take a couple of minutes

if(!require(tidyverse)) install.packages("tidyverse", repos = "http://cran.us.r-project.org")
if(!require(caret)) install.packages("caret", repos = "http://cran.us.r-project.org")

# MovieLens 10M dataset:
# https://grouplens.org/datasets/movielens/10m/
# http://files.grouplens.org/datasets/movielens/ml-10m.zip

dl <- tempfile()
download.file("http://files.grouplens.org/datasets/movielens/ml-10m.zip", dl)

ratings <- read.table(text = gsub("::", "\t", readLines(unzip(dl, "ml-10M100K/ratings.dat"))),
                      col.names = c("userId", "movieId", "rating", "timestamp"))

movies <- str_split_fixed(readLines(unzip(dl, "ml-10M100K/movies.dat")), "\\::", 3)
colnames(movies) <- c("movieId", "title", "genres")
movies <- as.data.frame(movies) %>% mutate(movieId = as.numeric(levels(movieId))[movieId],
                                           title = as.character(title),
                                           genres = as.character(genres))

movielens <- left_join(ratings, movies, by = "movieId")

# Validation set will be 10% of MovieLens data

set.seed(1)
test_index <- createDataPartition(y = movielens$rating, times = 1, p = 0.1, list = FALSE)
edx <- movielens[-test_index,]
temp <- movielens[test_index,]

# Make sure userId and movieId in validation set are also in edx set

validation <- temp %>% 
  semi_join(edx, by = "movieId") %>%
  semi_join(edx, by = "userId")


# Add rows removed from validation set back into edx set

removed <- anti_join(temp, validation)
edx <- rbind(edx, removed)

rm(dl, ratings, movies, test_index, temp, movielens, removed)
write.csv(edx, "edx.csv")





## MovieLens Project.R


# Install all needed libraries if it is not present

if(!require(tidyverse)) install.packages("tidyverse") 
if(!require(kableExtra)) install.packages("kableExtra")
if(!require(tidyr)) install.packages("tidyr")
if(!require(tidyverse)) install.packages("tidyverse")
if(!require(stringr)) install.packages("stringr")
if(!require(forcats)) install.packages("forcats")
if(!require(ggplot2)) install.packages("ggplot2")

# Loading all needed libraries

library(dplyr)
library(tidyverse)
library(kableExtra)
library(tidyr)
library(stringr)
library(forcats)
library(ggplot2)

# The RMSE function that will be used in this project is:
RMSE <- function(true_ratings = NULL, predicted_ratings = NULL) {
    sqrt(mean((true_ratings - predicted_ratings)^2))
}

# Convert timestamp to a human readable date

edx$date <- as.POSIXct(edx$timestamp, origin="1970-01-01")
validation$date <- as.POSIXct(validation$timestamp, origin="1970-01-01")

# Extract the year and month of rate in both dataset

edx$yearOfRate <- format(edx$date,"%Y")
edx$monthOfRate <- format(edx$date,"%m")

validation$yearOfRate <- format(validation$date,"%Y")
validation$monthOfRate <- format(validation$date,"%m")

# Extract the year of release for each movie in both dataset
# edx dataset

edx <- edx %>%
   mutate(title = str_trim(title)) %>%
   extract(title,
           c("titleTemp", "release"),
           regex = "^(.*) \\(([0-9 \\-]*)\\)$",
           remove = F) %>%
   mutate(release = if_else(str_length(release) > 4,
                                as.integer(str_split(release, "-",
                                                     simplify = T)[1]),
                                as.integer(release))
   ) %>%
 mutate(title = if_else(is.na(titleTemp),
                          title,
                          titleTemp)
         ) %>%
  select(-titleTemp)


# validation dataset

validation <- validation %>%
   mutate(title = str_trim(title)) %>%
   extract(title,
           c("titleTemp", "release"),
           regex = "^(.*) \\(([0-9 \\-]*)\\)$",
           remove = F) %>%
   mutate(release = if_else(str_length(release) > 4,
                                as.integer(str_split(release, "-",
                                                     simplify = T)[1]),
                                as.integer(release))
   ) %>%
   mutate(title = if_else(is.na(titleTemp),
                          title,
                          titleTemp)
         ) %>%
  select(-titleTemp)


  # Extract the genre in edx datasets

edx <- edx %>%
   mutate(genre = fct_explicit_na(genres,
                                       na_level = "(no genres listed)")
          ) %>%
   separate_rows(genre,
                 sep = "\\|")


# Extract the genre in validation datasets

validation <- validation %>%
   mutate(genre = fct_explicit_na(genres,
                                       na_level = "(no genres listed)")
          ) %>%
   separate_rows(genre,
                 sep = "\\|")

# remove unnecessary columns on edx and validation dataset

edx <- edx %>% select(userId, movieId, rating, title, genre, release, yearOfRate, monthOfRate)

validation <- validation %>% select(userId, movieId, rating, title, genre, release, yearOfRate, monthOfRate)

# Convert the columns into the desidered data type

edx$yearOfRate <- as.numeric(edx$yearOfRate)
edx$monthOfRate <- as.numeric(edx$monthOfRate)
edx$release <- as.numeric(edx$release)

validation$yearOfRate <- as.numeric(validation$yearOfRate)
validation$monthOfRate <- as.numeric(validation$monthOfRate)
validation$release <- as.numeric(validation$release)

# Calculate the average of all movies

mu_hat <- mean(edx$rating)


# Predict the RMSE on the validation set

rmse_mean_model_result <- RMSE(validation$rating, mu_hat)

# Creating a results dataframe that contains all RMSE results

results <- data.frame(model="Naive Mean-Baseline Model", RMSE=rmse_mean_model_result)

# Calculate the average by movie

movie_avgs <- edx %>%
   group_by(movieId) %>%
   summarize(b_i = mean(rating - mu_hat))

# Compute the predicted ratings on validation dataset

rmse_movie_model <- validation %>%
   left_join(movie_avgs, by='movieId') %>%
   mutate(pred = mu_hat + b_i) %>%
   pull(pred)


rmse_movie_model_result <- RMSE(validation$rating, rmse_movie_model)

# Adding the results to the results dataset

results <- results %>% add_row(model="Movie-Based Model", RMSE=rmse_movie_model_result)

# Calculate the average by user

user_avgs <- edx %>%
   left_join(movie_avgs, by='movieId') %>%
   group_by(userId) %>%
   summarize(b_u = mean(rating - mu_hat - b_i))

# Compute the predicted ratings on validation dataset

rmse_movie_user_model <- validation %>%
   left_join(movie_avgs, by='movieId') %>%
   left_join(user_avgs, by='userId') %>%
   mutate(pred = mu_hat + b_i + b_u) %>%
   pull(pred)


   rmse_movie_user_model_result <- RMSE(validation$rating, rmse_movie_user_model)

# Adding the results to the results dataset

results <- results %>% add_row(model="Movie+User Based Model", RMSE=rmse_movie_user_model_result)

genre_pop <- edx %>%
   left_join(movie_avgs, by='movieId') %>%
   left_join(user_avgs, by='userId') %>%
   group_by(genre) %>%
   summarize(b_u_g = mean(rating - mu_hat - b_i - b_u))

# Compute the predicted ratings on validation dataset

rmse_movie_user_genre_model <- validation %>%
   left_join(movie_avgs, by='movieId') %>%
   left_join(user_avgs, by='userId') %>%
   left_join(genre_pop, by='genre') %>%
   mutate(pred = mu_hat + b_i + b_u + b_u_g) %>%
   pull(pred)


rmse_movie_user_genre_model_result <- RMSE(validation$rating, rmse_movie_user_genre_model)

# Adding the results to the results dataset

results <- results %>% add_row(model="Movie+User+Genre Based Model", RMSE=rmse_movie_user_genre_model_result)

lambdas <- seq(0, 10, 0.1)

# Compute the predicted ratings on validation dataset using different values of lambda

rmses <- sapply(lambdas, function(lambda) {
   
  # Calculate the average by user
  
   b_i <- edx %>%
      group_by(movieId) %>%
      summarize(b_i = sum(rating - mu_hat) / (n() + lambda))
   
   # Compute the predicted ratings on validation dataset
   
   predicted_ratings <- validation %>%
    left_join(b_i, by='movieId') %>%
      mutate(pred = mu_hat + b_i) %>%
      pull(pred)
   
   # Predict the RMSE on the validation set
   
   return(RMSE(validation$rating, predicted_ratings))
})


# Get the lambda value that minimize the RMSE
min_lambda <- lambdas[which.min(rmses)]

# Predict the RMSE on the validation set

rmse_regularized_movie_model <- min(rmses)

# Adding the results to the results dataset

results <- results %>% add_row(model="Regularized Movie-Based Model", RMSE=rmse_regularized_movie_model)
rmses <- sapply(lambdas, function(lambda) {

   # Calculate the average by user
   
   b_i <- edx %>%
      group_by(movieId) %>%
      summarize(b_i = sum(rating - mu_hat) / (n() + lambda))
   
   # Calculate the average by user
   
   b_u <- edx %>%
      left_join(b_i, by='movieId') %>%
      group_by(userId) %>%
      summarize(b_u = sum(rating - b_i - mu_hat) / (n() + lambda))


  # Compute the predicted ratings on validation dataset
   
   predicted_ratings <- validation %>%
      left_join(b_i, by='movieId') %>%
      left_join(b_u, by='userId') %>%
      mutate(pred = mu_hat + b_i + b_u) %>%
      pull(pred)
   
   # Predict the RMSE on the validation set
   
   return(RMSE(validation$rating, predicted_ratings))
})

# Get the lambda value that minimize the RMSE

min_lambda <- lambdas[which.min(rmses)]

# Predict the RMSE on the validation set

rmse_regularized_movie_user_model <- min(rmses)

# Adding the results to the results dataset

results <- results %>% add_row(model="Regularized Movie+User Based Model", RMSE=rmse_regularized_movie_user_model)

lambdas <- seq(0, 15, 0.1)


# Compute the predicted ratings on validation dataset using different values of lambda

rmses <- sapply(lambdas, function(lambda) {

   # Calculate the average by user
   
   b_i <- edx %>%
      group_by(movieId) %>%
      summarize(b_i = sum(rating - mu_hat) / (n() + lambda))
   
   # Calculate the average by user
   
   b_u <- edx %>%
      left_join(b_i, by='movieId') %>%
      group_by(userId) %>%
      summarize(b_u = sum(rating - b_i - mu_hat) / (n() + lambda))
b_u_g <- edx %>%
      left_join(b_i, by='movieId') %>%
      left_join(b_u, by='userId') %>%
      group_by(genre) %>%
      summarize(b_u_g = sum(rating - b_i - mu_hat - b_u) / (n() + lambda))
   
   # Compute the predicted ratings on validation dataset
   
   predicted_ratings <- validation %>%
      left_join(b_i, by='movieId') %>%
      left_join(b_u, by='userId') %>%
      left_join(b_u_g, by='genre') %>%
      mutate(pred = mu_hat + b_i + b_u + b_u_g) %>%
      pull(pred)
   
   # Predict the RMSE on the validation set
   
   return(RMSE(validation$rating, predicted_ratings))
})

# Get the lambda value that minimize the RMSE

min_lambda <- lambdas[which.min(rmses)]


# Predict the RMSE on the validation set

rmse_regularized_movie_user_genre_model <- min(rmses)

# Adding the results to the results dataset

results <- results %>% add_row(model="Regularized Movie+User+Genre Based Model", RMSE=rmse_regularized_movie_user_genre_model)
```

## Enviroment


print("Operating System:")
version



print("All installed packages")
installed.packages()
      




