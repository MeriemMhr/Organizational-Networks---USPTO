---
title: "Project_Code"
output:
  pdf_document: 
    toc: true
    number_sections: true
    latex_engine: xelatex
  always_allow_html: true
date: "2024-04-20"
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Load necessary libraries for data manipulation, visualization, and analysis

```{r}
# Loading necessary libraries
#install.packages('ggthemes')
#install.packages('ggridges')
#install.packages('webshot2')
library(arrow)       # For reading/writing Apache Parquet files
library(gender)      # For predicting gender based on first names
library(dplyr)       # For data manipulation
library(tidyr)       # For tidying data
library(wru)         # For predicting race
library(lubridate)   # For working with dates
library(ggplot2)     # For data visualization
library(ggthemes)    # For additional ggplot themes
library(lattice)     # For additional plotting options
library(tidyverse)   # For a cohesive data analysis workflow
library(ggridges)    # For creating ridge plots
library(tidygraph)   # For graph data structures
library(ggraph)      # For graph visualization
library(webshot2)     # For taking html widgets snapshots to knit in PDF
```

## Load Data

### To begin the analysis, we load our primary datasets: patent applications stored in a Parquet file and a CSV file containing edges that represent relationships between patent examiners. This step sets the foundation for our examination of the USPTO's patent examination process.

```{r}
# Define the path to the data directory
data_path <- "E:/Users/pc/Downloads/672_project_data/"
applications <- read_parquet(paste0(data_path,"app_data_sample.parquet"))
edges <- read_csv(paste0(data_path,"edges_sample.csv"))

```

## Reveiwing the parquet file

```{r}
head(applications)
```

## Reviewing the csv file

```{r}
head(edges)
```

## Get gender for examiners

### We’ll get gender based on the first name of the examiner, which is recorded in the field examiner_name_first. We’ll use library gender for that, relying on a modified version of their own example.

###Note that there are over 2 million records in the applications table – that’s because there are many records for each examiner, as many as the number of applications that examiner worked on during this time frame. Our first step therefore is to get all unique names in a separate list examiner_names. We will then guess gender for each one and will join this table back to the original dataset. So, let’s get names without repetition:

```{r}
# get a list of first names without repetitions
examiner_names <- applications %>% distinct(examiner_name_first)

head(examiner_names)
```

### Now let’s use function gender() as shown in the example for the package to attach a gender and probability to each name and put the results into the table examiner_names_gender

```{r}
# Use the gender package to estimate gender based on first names
examiner_names_gender <- examiner_names %>% 
  do(results = gender(.$examiner_name_first, method = "ssa")) %>% 
  unnest(cols = c(results), keep_empty = TRUE) %>%
  select(
    examiner_name_first = name,
    gender,
    proportion_female
  ) %>% 
  # Filter out rows where any of the specified columns are NA
  filter(!is.na(gender))

print(head(examiner_names_gender))
```

### Finally, let’s join that table back to our original applications data and discard the temporary tables we have just created to reduce clutter in our environment.

```{r}
# remove extra colums from the gender table
examiner_names_gender <- examiner_names_gender %>% 
  select(examiner_name_first, gender)

# joining gender back to the dataset
applications <- applications %>% 
  left_join(examiner_names_gender, by = "examiner_name_first")

# cleaning up
rm(examiner_names)
rm(examiner_names_gender)
gc()
```

## Guess the examiner’s race

### We’ll now use package wru to estimate likely race of an examiner. Just like with gender, we’ll get a list of unique names first, only now we are using surnames.

```{r}
# Isolate unique last names for race prediction
examiner_surnames <- applications %>% 
  select(surname = examiner_name_last) %>% 
  distinct()

head(examiner_surnames)
```

### We’ll follow the instructions for the package outlined here https://github.com/kosukeimai/wru.

```{r}
# Use the wru package to estimate race based on surnames
examiner_race <- examiner_surnames %>%
  # Ensure we're working with clean, non-NA surnames
  filter(!is.na(surname)) %>%
  # Apply the race prediction
  predict_race(voter.file = ., surname.only = TRUE) %>% 
  as_tibble()
```

### Exploring examiner_race

```{r}
head(examiner_race)
```

### As you can see, we get probabilities across five broad US Census categories: white, black, Hispanic, Asian and other. (Some of you may correctly point out that Hispanic is not a race category in the US Census, but these are the limitations of this package.)

### Our final step here is to pick the race category that has the highest probability for each last name and then join the table back to the main applications table. See this example for comparing values across columns: https://www.tidyverse.org/blog/2020/04/dplyr-1-0-0-rowwise/. And this one for case_when() function: https://dplyr.tidyverse.org/reference/case_when.html.

```{r}
# Determine the most likely race category for each surname
examiner_race <- examiner_race %>% 
  mutate(max_race_p = pmax(pred.asi, pred.bla, pred.his, pred.oth, pred.whi)) %>% 
  mutate(race = case_when(
    max_race_p == pred.asi ~ "Asian",
    max_race_p == pred.bla ~ "black",
    max_race_p == pred.his ~ "Hispanic",
    max_race_p == pred.oth ~ "other",
    max_race_p == pred.whi ~ "white",
    TRUE ~ NA_character_
  ))

head(examiner_race)
```

### Let’s join the data back to the applications table.

```{r}
# removing extra columns
examiner_race <- examiner_race %>% 
  select(surname,race)

# Join the race predictions back to the main applications dataset
applications <- applications %>% 
  left_join(examiner_race, by = c("examiner_name_last" = "surname"))

# Again, clean up the workspace by removing temporary variables
rm(examiner_race)
rm(examiner_surnames)
gc()
```
## Examiner’s tenure

### To figure out the timespan for which we observe each examiner in the applications data, let’s find the first and the last observed date for each examiner. We’ll first get examiner IDs and application dates in a separate table, for ease of manipulation. We’ll keep examiner ID (the field examiner_id), and earliest and latest dates for each application (filing_date and appl_status_date respectively). We’ll use functions in package lubridate to work with date and time values.

```{r}
# Extract relevant date information for each application
examiner_dates <- applications %>% 
  select(examiner_id, filing_date, appl_status_date) 

head(examiner_dates)
```

### The dates look inconsistent in terms of formatting. Let’s make them consistent. We’ll create new variables start_date and end_date.

```{r}
# Standardize date formats and calculate tenure
examiner_dates <- examiner_dates %>% 
  mutate(start_date = ymd(filing_date), end_date = as_date(dmy_hms(appl_status_date)))
```

### Let’s now identify the earliest and the latest date for each examiner and calculate the difference in days, which is their tenure in the organization.

```{r}
# Calculate the tenure for each examiner based on the earliest and latest dates observed
examiner_tenure <- examiner_dates %>%
  # Remove rows with NA in start_date or end_date before grouping and summarising
  filter(!is.na(start_date) & !is.na(end_date)) %>%
  group_by(examiner_id) %>%
  summarise(
    earliest_date = min(start_date, na.rm = TRUE), 
    latest_date = max(end_date, na.rm = TRUE),
    tenure_days = interval(earliest_date, latest_date) %/% days(1),
    .groups = 'drop' # Automatically drop the grouping
  ) %>% 
  # Keep records with a latest_date before 2018
  filter(year(latest_date) < 2018)

# Assuming you want to check the result
head(examiner_tenure)
```

### Joining back to the applications data.

```{r}
applications <- applications %>% 
  left_join(examiner_tenure, by = "examiner_id")

rm(examiner_tenure)
gc()
```
### Review the applications dataframe after merging examiner_tenure

```{r}
head(applications)
```

### Adding workgroup column to the applications dataframe to proceed with the analysis

```{r}
applications <- applications %>%
  mutate(workgroup = substr(examiner_art_unit, 1, 3))
```

### Removing missing values for further analysis

```{r}
applications_data <- applications %>%
  filter(!is.na(gender) & !is.na(race) & !is.na(examiner_id))

# Assuming you want to check the result
print(head(applications_data))
```

### Analyzing Demographics and Tenure in Selected Workgroups

### The next phase of the analysis focuses on understanding the demographics (gender and race) and tenure within specific workgroups. This involves comparing these attributes across different workgroups to uncover any patterns or disparities that might exist.

```{r}
# Set seed for reproducibility
set.seed(123)

# Assuming 'applications' dataframe has a column 'examiner_art_unit'
# Extract unique workgroups
# This approach ensures a focused examination on a subset, making the analysis more manageable and targeted
unique_workgroups <- unique(substring(applications_data$examiner_art_unit, 1, 3))

# Randomly sample 2 unique workgroups
sampled_workgroups <- sample(unique_workgroups, 2)

# Filter applications for only those workgroups
applications_filtered <- applications_data %>%
  mutate(workgroup = substring(examiner_art_unit, 1, 3)) %>%
  filter(workgroup %in% sampled_workgroups)

# View the selected workgroups
print(sampled_workgroups)
```

### At this point, applications_filtered contains data for a focused group of examiners. This subset will be used for detailed demographic analysis and tenure examination, providing insights into these specific workgroups.


```{r}
# Prepare data specifically for the selected workgroups (e.g., 247 and 216) for comparison
# Convert examiner_art_unit to character to ensure string operations work correctly
app_data_sample <- applications_data %>%
  mutate(examiner_art_unit = as.character(examiner_art_unit))

# Filter for workgroups 247 and 216
selected_workgroups <- app_data_sample %>%
  filter(str_starts(examiner_art_unit, "247") | str_starts(examiner_art_unit, "216"))
```

### Calculate summary statistics and create visualizations to compare demographics and tenure within the selected workgroups. This step aims to uncover any notable trends or differences that could inform organizational strategies or highlight areas for further investigation

```{r}
# Summary Statistics and Plots
# Summary statistics for gender, race, and tenure
# Compute average tenure days and count by workgroup, gender, and race
summary_stats <- selected_workgroups %>%
  group_by(substring(examiner_art_unit, 1, 3), gender, race) %>%
  summarise(
    avg_tenure_days = mean(tenure_days, na.rm = TRUE),
    n = n(),
    .groups = 'drop'
  )

print(summary_stats)
```


### Visualization of Demographics. 
### Generate plots to visually compare the gender and race distribution within the selected workgroups
### These visualizations help in quickly identifying disparities and patterns

```{r}
# Plots for demographics
# Gender distribution
gender_dist_plot <- ggplot(selected_workgroups, aes(x = substring(examiner_art_unit, 1, 3), fill = gender)) +
  geom_bar(position = "dodge") +
  labs(title = "Gender Distribution in Selected Workgroups", x = "Workgroup", y = "Count")

# Race distribution across workgroups
race_dist_plot <- ggplot(selected_workgroups, aes(x = substring(examiner_art_unit, 1, 3), fill = race)) +
  geom_bar(position = "dodge") +
  labs(title = "Race Distribution in Selected Workgroups", x = "Workgroup", y = "Count")

print(gender_dist_plot)
print(race_dist_plot)
```

Workgroup "216" appears to have a greater racial diversity compared to "247", which is predominantly composed of one racial group (perhaps 'white', though the labels are not visible in the data provided).
The presence of the 'Asian' and 'Hispanic' categories in noticeable but smaller proportions suggests some level of diversity within the workgroups.
 Understanding these distributions is crucial for the organization's diversity and inclusion efforts and might warrant further investigation into recruitment and retention practices.


### These plots provide a clear visual representation of gender and race distributions within the selected workgroups, making it easier to identify any imbalances or diversity issues that may require attention.


```{r}
# Create a new column 'workgroup_prefix' to store the prefix for coloring
cleaned_filtered_workgroups <- selected_workgroups %>%
  mutate(workgroup_prefix = substr(examiner_art_unit, 1, 3))

# Now, plot the density of tenure_days colored by 'workgroup_prefix'
ggplot(cleaned_filtered_workgroups, aes(x = tenure_days, fill = workgroup_prefix)) +
  geom_density(alpha = 0.7) +
  scale_fill_manual(values = c("247" = "#66C2A5", "216" = "#FC8D62"), name = "Workgroup") +
  labs(title = "Density Plot of Examiner Tenure Days for Selected Workgroups",
       x = "Tenure Days", y = "Density",
       caption = "Data Source: Examiner Applications") +
  theme_light(base_size = 14) +
  theme(plot.title = element_text(face = "bold", size = 12),
        axis.title = element_text(face = "bold"),
        legend.position = "right")
```


The density plot provides a view of the distribution of tenure days:

There's a significant peak in tenure for workgroup "247" suggesting a cluster of examiners with a similar, high number of tenure days.
Workgroup "216" shows a more evenly spread distribution with several smaller peaks, indicating a more varied range of tenure days.
Such patterns may reflect the history and turnover rates within the workgroups and could be used to plan for future workforce needs or retirements.

### Enhancing Visualization for In-depth Analysis

### After preparing our data with gender, race, and tenure information, we focus on visualizing the tenure distribution within our selected workgroups. This is crucial for understanding the diversity and experience within these groups.

## Violin Plot

### To delve deeper into the tenure distribution by combining the density plot with a box plot for each workgroup. This hybrid visualization provides a comprehensive view of the tenure distribution, including median tenure and variability.
```{r}
# Violin Plot for Tenure Days by Workgroup
ggplot(applications_filtered, aes(x = workgroup, y = tenure_days, fill = workgroup)) +
  geom_violin(trim = FALSE) +
  geom_boxplot(width = 0.1, fill = "white") +
  scale_fill_manual(values = c("#66C2A5", "#FC8D62"), name = "Workgroup") +
  labs(title = "Violin Plot of Examiner Tenure Days by Workgroup",
       x = "Workgroup",
       y = "Tenure Days",
       caption = "Data Source: Examiner Applications") +
  theme_light(base_size = 14) +
  theme(plot.title = element_text(face = "bold", size = 16),
        axis.title = element_text(face = "bold"),
        legend.position = "none")
```

The violin plot offers a visual summary of tenure distribution:

Both workgroups have a wide range of tenure days, but "247" shows a more pronounced concentration at the higher end of tenure.
The width of the violins at different tenure day levels indicates the density of examiners at that experience level.
Insights from this plot can inform decisions about potential skill gaps, mentorship opportunities, and the allocation of new or complex cases based on examiner experience.


## Scatter plot

### To evaluate the potential correlation between examiners' tenure and their productivity across two specific workgroups, "247" and "216". This visualization aims to discern patterns that could indicate whether more experienced examiners have higher or lower workloads, and to what extent tenure influences application handling capacity.

## Density Ridge Plot: Distribution of Tenure Days by Workgroup

### The aim is to visualize and compare the distribution of tenure across examiners in the selected workgroups, "247" and "216". The density ridge plot seeks to highlight the commonality of tenure lengths and the spread of experience within these groups.


```{r}
library(ggplot2)

#Scatter plot
applications_filtered %>%
  filter(workgroup %in% c("247", "216")) %>%
  group_by(workgroup, examiner_id) %>%
  summarise(tenure_days = mean(tenure_days, na.rm = TRUE), 
            app_count = n(), .groups = 'drop') %>%
  ggplot(aes(x = tenure_days, y = app_count, color = workgroup)) +
  geom_point(alpha = 0.6) +
  labs(title = "Relationship between Tenure Days and Application Count by Workgroup",
       x = "Average Tenure Days", 
       y = "Number of Applications Handled") +
  theme_minimal() +
  scale_color_manual(values = c("247" = "#66C2A5", "216" = "#FC8D62"))


#Distribution of tenure days by workgroup
applications_filtered %>%
  filter(workgroup %in% c("247", "216")) %>%
  ggplot(aes(x = tenure_days, y = factor(workgroup), fill = workgroup)) +
  geom_density_ridges(scale = 0.9) +
  labs(title = "Distribution of Tenure Days by Workgroup",
       x = "Tenure Days",
       y = "Workgroup") +
  scale_fill_manual(values = c("247" = "#66C2A5", "216" = "#FC8D62")) +
  theme_ridges(grid = TRUE) +
  theme(legend.position = "right")
```

Relationship between Tenure Days and Application Count by Workgroup

The scatter plot relating tenure days to application count yields several insights:

There doesn't appear to be a clear, linear relationship between tenure days and the number of applications handled within each workgroup.
Some individuals with a high number of tenure days handle a large number of applications, which might indicate a correlation between experience and productivity.
Notably, there are examiners with fewer tenure days handling a high volume of applications. This could point to efficient training programs or possibly to overburdening of less experienced staff.

Finally, the distribution chart showcases the tenure days for the two workgroups:

Workgroup "247" shows a substantial peak around 6000 tenure days, which might indicate the presence of a cohort hired around the same time or a retention pattern.
Workgroup "216" has a more uniform distribution but with fewer individuals reaching the highest tenure days seen in "247".
This chart can highlight tenure-related dynamics, such as the potential for knowledge loss if a retiring cohort leaves simultaneously or the readiness for leadership roles within the groups.


## Examining Advice Networks and Centrality:

### To understand the social and advisory structures within the organization by mapping how examiners interact within selected workgroups.



```{r}
library(igraph)

## Step 3: Create Advice Networks and Calculate Centrality Scores
# Convert examiner_id in edges to character to match types
edges <- edges %>%
  mutate(ego_examiner_id = as.character(ego_examiner_id),
         alter_examiner_id = as.character(alter_examiner_id))

# Convert examiner_id in selected_workgroups to character to match the types in edges
selected_workgroups <- selected_workgroups %>%
  mutate(examiner_id = as.character(examiner_id))

# Now perform the join with matching types
selected_edges <- edges %>%
  inner_join(selected_workgroups %>% select(examiner_id), by = c("ego_examiner_id" = "examiner_id")) %>%
  select(ego_examiner_id, alter_examiner_id)

# Filter edges for selected workgroups by joining with the selected workgroups
# Assuming that examiner_id is already a character in selected_workgroups
selected_edges <- edges %>%
  inner_join(selected_workgroups %>% select(examiner_id), by = c("ego_examiner_id" = "examiner_id")) %>%
  select(ego_examiner_id, alter_examiner_id)

# Create advice networks
g <- graph_from_data_frame(selected_edges, directed = TRUE)

# Calculate centrality scores (e.g., degree centrality)
degree_centrality <- degree(g, mode = "in")
betweenness_centrality <- betweenness(g)

# Associate centrality scores with examiners
centrality_scores <- data.frame(
  examiner_id = V(g)$name,
  degree = degree_centrality,
  betweenness = betweenness_centrality
)

# Make sure examiner_id is a character in both data frames before joining
selected_workgroups <- selected_workgroups %>%
  mutate(examiner_id = as.character(examiner_id))

centrality_scores <- centrality_scores %>%
  mutate(examiner_id = as.character(examiner_id))
```

### To analyze how centrality within the advice network correlates with demographic factors and tenure, exploring potential patterns of influence and interaction dynamics.

```{r}
## Step 4: Analyze Relationship Between Centrality and Examiner Demographics
# Merge centrality scores with demographic data
analysis_data <- selected_workgroups %>%
  inner_join(centrality_scores, by = "examiner_id")

# Example analysis: Correlation between tenure and centrality
cor_analysis <- cor.test(analysis_data$tenure_days, analysis_data$degree, use = "complete.obs")

print(cor_analysis)
```

## Network Visualization:

### To visually represent the advice networks, highlighting key individuals based on their centrality measures.

```{r}
# Join centrality scores with the selected workgroups data
workgroups_with_centrality <- left_join(selected_workgroups, centrality_scores, by = "examiner_id")

```

### To visually represent the advice networks, highlighting key individuals based on their centrality measures. (Was taking a lot of time, so commented it out)

### The visualization makes it easier to identify the structure of the network and the positions of key examiners. It allows us to see how well-connected the network is, the existence of clusters or communities within the workgroup, and whether certain individuals act as bridges or hubs.

```{r}
# Associate centrality scores with the nodes in the graph
#V(g)$degree <- centrality_scores$degree
#V(g)$betweenness <- centrality_scores$betweenness

# Plotting the network with ggraph
#ggraph(g, layout = "fr") + 
  #geom_edge_link(color = "gray", alpha = 0.5) +  # Draw edges
  #geom_node_point(aes(size = degree, color = betweenness), alpha = 0.9) +  # Nodes colored by #betweenness and sized by degree
  #scale_color_viridis_c() +  # Use Viridis color scale for betweenness
  #theme_void() +  # Remove background and axes for a clean look
  #ggtitle("Network Visualization with Centrality Measures") +  # Add title
  #labs(color = "Betweenness Centrality", size = "Degree Centrality")  # Label legends

```


Given the correlation results between tenure days and the degree centrality measure (correlation coefficient = 0.137, highly significant with p-value < 2.2e-16), here’s an analytical discussion on the choice of centrality measures and their relationship with examiner characteristics:

Choice of Centrality Measure

1. Degree Centrality:
Justified by the correlation results, degree centrality (which measures the number of direct connections an examiner has) is a good starting point because it gives a straightforward indication of how active an examiner is in the advice network. An examiner with high degree centrality is likely someone who either seeks a lot of advice or is sought after for advice by many colleagues.

Pros: Simple to calculate and interpret; immediately indicates active network participants.
Cons: Does not account for the indirect influence or the hierarchical structure of the network.

2. Betweenness Centrality:
This would be another excellent measure to consider. It captures an individual's role as an intermediary in the communication flow within the network. An examiner with high betweenness centrality would be someone who connects different clusters within the network, potentially serving as a gatekeeper or bridge of information.

Pros: Highlights individuals who control information flow and connect disparate groups.
Cons: More computationally intensive and can be less intuitive to interpret.

Relationship Between Centrality and Examiners’ Characteristics

The analysis of the correlation between tenure days and degree centrality suggests a positive relationship, although it is relatively weak (correlation coefficient around 0.137). Here’s what this relationship might indicate:

Mild Positive Relationship: Examiners with longer tenure may have slightly more connections within the network, possibly due to having had more time to establish relationships. However, the relationship is not strong, which suggests that factors other than tenure also play a significant role in an examiner's network centrality.
Centrality as a Function of Multiple Factors: Other characteristics, such as job performance, communication skills, and position within the organization, may also significantly influence centrality. An examiner's expertise, area of specialty, and willingness to share knowledge could contribute to their central role in the advice network.
Centrality and Influence: Examiners with higher centrality, particularly those with high betweenness centrality, may have considerable influence over the spread of information and decision-making processes within the organization.

## Processing Time Calculation

### A crucial aspect of our analysis is the calculation of application processing time, the duration from filing to the final decision. This step is vital for understanding efficiency within the patent examination process.

```{r}
# Dropping applications with "Pending" status to focus on completed cases
applications <- applications %>%
  filter(disposal_type != "PEND")

# Calculating application processing time
applications <- applications %>%
  mutate(app_proc_time = interval(
    ymd(filing_date),
    dmy_hms(appl_status_date)
  ) %/% days(1))

# Final cleanup
gc()

# Previewing the updated dataset
head(applications)
```
## Preparing Edge Data

### This code block transforms the edges dataframe, which contains relationships between patent examiners, by ensuring the examiner IDs (both ego_examiner_id and alter_examiner_id) are character strings. This is crucial for graph analysis as it ensures consistency in node identification. We also drop any rows with missing values to maintain data integrity.

```{r}
edges <- edges %>%
  mutate(
    from = as.character(ego_examiner_id), # Convert IDs to character for graph compatibility
    to = as.character(alter_examiner_id)
  ) %>%
  drop_na() # Remove rows with missing values
```


## Creating a Network Graph

### We then prepare the applications dataframe for integration into the network graph. This includes relocating examiner_id for easier access, converting IDs to character strings for consistency with edge data, and renaming examiner_id to name for clarity. A directed graph is created from the edges dataframe, incorporating examiner data from applications.


```{r}
# Preparing applications data for graph creation
applications <- applications %>%
  relocate(examiner_id, .before = application_number) %>%
  mutate(examiner_id = as.character(examiner_id)) %>%
  drop_na(examiner_id) %>%
  rename(name = examiner_id)

# Creating a directed graph from the edges data
graph <- tbl_graph(
  edges = (edges %>% relocate(from, to)),
  directed = TRUE
)

# Enriching graph nodes with examiner data from applications
graph <- graph %>%
  activate(nodes) %>%
  inner_join(
    (applications %>% distinct(name, .keep_all = TRUE)),
    by = "name"
  )

# Display the graph structure
graph
```

The network comprises 2,489 nodes and 17,720 edges, characterizing a directed multigraph with significant interactions among patent examiners at the USPTO. The designation as a "directed multigraph" is particularly telling, indicating not only the directional nature of these interactions (suggesting a flow of communication or influence from one examiner to another) but also the presence of multiple connections between the same pairs of examiners. 

This could reflect recurring collaborations or consultations on various patent applications, highlighting a complex web of professional relationships. The existence of 127 distinct components within this network suggests a segmented operational structure, where clusters of examiners may work more closely with each other, potentially aligned by specialization areas or other organizational divisions. This fragmentation could mirror the diverse technical fields covered by patent applications, indicating a natural grouping of examiners based on expertise or departmental organization.


```{r}

# Plotting the network with ggraph
ggraph(graph, layout = "fr") + 
  geom_edge_link(color = "gray", alpha = 0.5) +  # Draw edges
  geom_node_point(alpha = 0.9) +  # Nodes colored by betweenness and sized by degree
  scale_color_viridis_c() +  # Use Viridis color scale for betweenness
  theme_void() +  # Remove background and axes for a clean look
  ggtitle("Network Visualization with Centrality Measures") +  # Add title
  labs(color = "Betweenness Centrality", size = "Degree Centrality")  # Label legends
```

Central Node Cluster: The most striking feature is the central mass of nodes, which indicates a highly interconnected group. These could represent a core group of examiners or departments within the USPTO that have strong interrelations, perhaps due to shared responsibilities, similar expertise, or frequent collaboration. The density of this cluster suggests robust communication paths that could facilitate the flow of information and potentially expedite the patent examination process.
Outlying Nodes: The scattered, less connected nodes around the periphery of the network could represent examiners or units with more specialized roles or those less integrated into the main communication channels. These nodes might experience and contribute to longer processing times due to fewer collaborative connections or more specialized knowledge that is not as widely shared within the network.
Centrality Measures: Assuming the visualization takes into account centrality measures, the nodes at the center of the network likely have higher centrality, indicating their importance in the network's connectivity. These key nodes, possibly senior examiners or those in central administrative roles, may influence the pace and outcomes of the patent examination process significantly.
Implications for Efficiency and Equity: The network structure's implications for examination times might hinge on how well-connected the overall network is and how accessible central nodes are for peripheral ones. Moreover, if connectedness correlates with demographic characteristics, it may indicate potential inequities in work distribution or access to information, which could have further implications for the timing and outcomes of patent examinations.
In terms of connectedness, the central cluster would be the main area of focus to understand the dynamics of patent examination processes. The connectivity of these central nodes to the periphery might highlight pathways for potentially increasing efficiency and addressing any discrepancies in examination times that could be associated with network structure.


## Calculating Centrality Measures

### After constructing the network graph, we calculate centrality measures (degree, betweenness, closeness) for each node (examiner) to assess their influence within the network. These measures are then arranged by degree to highlight the most central examiners. The result is converted to a tibble for easy viewing and manipulation.

```{r}
node_data <- graph %>%
  activate(nodes) %>%
  mutate(
    degree = centrality_degree(),
    betweenness = centrality_betweenness(),
    closeness = centrality_closeness()
  ) %>%
  arrange(-degree) %>%
  as_tibble() %>%
  mutate(tc = as.factor(tc))

head(node_data)
```

## Regression Analysis with Plotting

### Finally, the run_regression function is designed to perform linear regression analysis based on specified predictor (x) and response (y) variables. If desired, it also generates a scatter plot with a regression line and saves it as a PNG file. This function simplifies the process of examining relationships between variables, such as the effect of centrality measures on application processing times.


```{r}
# Function to run regression and optionally generate and save a plot
run_regression <- function(data, x, y, plot = TRUE) {
  # Construct the regression formula
  formula <- as.formula(paste(y, "~", x))
  
  # Fitting the linear model
  model <- lm(formula, data = data)
  
  # Conditionally generate and save the plot
  if (plot) {
    # Prepare and display the plot
    plot_data <- ggplot(data, aes_string(x, y)) +
      geom_point() +
      geom_smooth(method = "lm", se = FALSE) +
      labs(title = paste("Regression of", y, "on", x),
           subtitle = paste("Adjusted R-squared:", round(summary(model)$adj.r.squared, 4)),
           x = x, y = y) +
      theme_minimal() +
      theme(plot.title = element_text(size = 16, face = "bold"),
            plot.subtitle = element_text(size = 14)) +
      labs(caption = "Source: USPTO Data")
    
    # Save the plot to a specified path
    ggsave(paste0("E:/", y, "_on_", x, ".png"), plot_data, width = 16, height = 9)
    
    # Display the plot
    print(plot_data)
  }
  
  # Extract model summary statistics
  tidy_model <- broom::tidy(model)
  glance_model <- broom::glance(model)
  
  # Enhance the summary dataframe with R-squared and centrality measure
  tidy_model <- tidy_model %>%
    mutate(adj_r_squared = glance_model$adj.r.squared, #Used adjusted r-squared value 
           centrality_measure = x)
  
  # Return the enhanced summary dataframe
  return(tidy_model)
}
```

##Running Regression Models on Centrality Measures


### Initiating a regression analysis exploring how the degree centrality of examiners correlates with application processing times. Degree centrality represents the number of direct connections an examiner has within the network, serving as a proxy for their involvement and potential influence in the patent examination process. The expectation is that examiners with higher degree centrality might expedite or delay the processing due to their central role in the collaborative network.

```{r}
# Running regression with Degree Centrality as the predictor for Application Processing Time
run_regression(node_data, "degree", "app_proc_time")
```

Observation:
Minimal Influence of Degree Centrality: The Adjusted R-squared value is very low at 0.00168, indicating that degree centrality accounts for a marginal amount of the variability in patent application processing times. This suggests that the number of direct connections an examiner has within the USPTO network has a negligible impact on how quickly they process applications.
Statistically Significant Yet Small Effect: The coefficient for degree centrality is small but positive (around 4.087), and with a p-value of approximately 0.023, it is statistically significant. This implies that a higher degree of centrality correlates with a minor increase in processing time. The relationship is significant in the statistical sense but not necessarily meaningful in a practical context due to the very small effect size.
Interpretation of Intercept: A significant intercept value of approximately 2468.65 suggests a baseline processing time for applications, likely reflective of the inherent time required for processing independent of the examiner’s network centrality.
Need for a Broader Model: Given the small explanatory power of degree centrality, it is evident that other factors beyond the examiner's immediate network influence processing times. Factors potentially missing from this model could include the complexity of patent applications, the examiner's specific expertise, workload distribution, and the nature of the technology involved in the applications.
Implications for the USPTO: The results highlight a need for the USPTO to consider a wider range of variables when looking to understand and improve processing efficiency. The structural position of an examiner, while statistically relevant, does not offer substantial insights into processing durations and hence should not be the sole focus of efficiency improvement strategies.


### Shifting focus to betweenness centrality, measuring an examiner's capacity to act as a bridge within the network. This analysis seeks to understand if examiners who frequently connect disparate groups impact the efficiency of patent processing differently, potentially by facilitating knowledge transfer or creating bottlenecks.

```{r}
# Running regression with Betweenness Centrality as the predictor for Application Processing Time
run_regression(node_data, "betweenness", "app_proc_time")

```

Statistical Insignificance of Betweenness: The betweenness centrality, despite being a theoretically interesting measure of an examiner's influence in the network, shows a non-significant p-value of approximately 0.575, suggesting that its role as a connector within the USPTO network does not have a discernible impact on processing times.
Negligible Adjusted R-squared: The model's Adjusted R-squared is negative, practically rounding to zero, which underscores that the model, including betweenness centrality, does not meaningfully improve upon a baseline model without predictors. This suggests that betweenness centrality may not be the right metric for understanding processing efficiency in this context.
Intercept Interpretation: The significant intercept indicates a baseline processing time, which is high, suggesting there is a considerable standard processing duration for applications that is not accounted for by the network structure captured by betweenness centrality.
Reevaluation of Predictor Utility: The lack of a significant association with processing times implies that the utility of betweenness centrality for predicting processing efficiency is limited. This finding challenges the assumption that individuals who act as bridges in the communication network necessarily process applications more or less efficiently.
Call for Additional Research: The results suggest that other factors may more significantly influence processing times. Future research should consider different variables, such as individual examiner workload, the complexity of patent applications, or the internal policies and support structures of the USPTO.
Broader Network Dynamics: The minimal impact of betweenness centrality on processing times may indicate that the efficiency of patent processing is less about individual examiner's network positions and more about the broader dynamics of the examination process or organizational structure. It could be beneficial to investigate how collaborative efforts, knowledge sharing, and resource allocation within the USPTO affect the speed of processing.


### Closeness centrality assesses how close an examiner is to all other nodes in the network, indicative of their accessibility. This regression model probes whether examiners who can more readily access or be accessed by others influence application processing times, perhaps by being more efficient in information dissemination or coordination.

```{r}
# Running regression with Closeness Centrality as the predictor for Application Processing Time
run_regression(node_data, "closeness", "app_proc_time")
```


Observations:
Intercept Insight: The positive and significant intercept suggests that the baseline processing time is substantial when all other variables are at their reference levels or absent. This baseline likely reflects the intrinsic administrative duration required for all applications, regardless of examiner-specific factors.
Ineffectual Closeness Centrality: The closeness centrality's coefficient is not statistically significant (p ≈ 0.857), affirming that this network measure doesn't notably influence the processing time of patent applications. This insignificance is reflected in the negligible and slightly negative Adjusted R-squared value, indicating that including closeness centrality in the model does not enhance its explanatory power and may, in fact, detract from it due to model complexity.
Model Reassessment: The model's inadequacy in explaining processing times via closeness centrality alone highlights the need for a broader assessment. This might involve incorporating more direct measures of an examiner's workload, the nature of patent applications, or other organizational factors that closeness centrality does not capture.
Centrality's Limited Role: Given the non-significant relationship, the model suggests that an examiner's accessibility within their network, at least as defined by closeness centrality, does not equate to notable differences in efficiency or administrative speed. This insight invites a focus on alternative influences that might better predict processing durations, such as the complexity of the patent application or the operational workflows within the USPTO.
Future Research Directions: The absence of a significant association between processing times and closeness centrality calls for further research. Such work could involve exploring additional network metrics, examining the specific characteristics of patent applications, or delving into the resource availability and organizational support that could facilitate or hinder an examiner's workflow.


## Advanced Regression Analysis Incorporating Demographic Interactions

### Extending the analysis by introducing interaction terms between centrality measures and demographic variables (gender and race). It systematically examines how the relationship between centrality and processing times may vary across different demographic groups, leveraging the map_dfr function for efficient iteration across centrality types. This approach acknowledges the complexity of the examiner network, considering the multifaceted influences on patent processing times.

```{r}
# Conducting regression analyses to explore the interactions between centrality measures, gender, and race on Application Processing Time
centrality_measures <- c("degree", "betweenness", "closeness")

results_df <- map_dfr(
  centrality_measures,
  ~ run_regression(node_data,
    paste0(.x, " * gender * race"),
    "app_proc_time",
    plot = FALSE
  )
)

# Extracting and displaying the R-squared values for each model to assess their explanatory power
results_df %>%
  select(centrality_measure, adj_r_squared) %>%
  distinct()
```

Marginal Explanatory Power: The Adjusted R-squared values are extremely low, with the highest being 0.00054 for the model including the interaction of degree centrality with gender and race. This indicates that these factors together account for less than 0.1% of the variance in application processing times, suggesting they have negligible predictive power.
Potential Overfitting Indication: The negative Adjusted R-squared values for the models incorporating betweenness and closeness centrality interactions imply that these models may actually fit worse than a simple horizontal line. A negative Adjusted R-squared can occur when the model is overfitted, suggesting that the inclusion of these interaction terms does not improve and may even detract from the model's ability to account for the variability in processing times.
Consideration of Model Relevance: The highest Adjusted R-squared value comes from the model that includes degree centrality interactions, although it remains near zero. While statistically higher than the other two, practically, it offers little to no enhancement in explaining the variance in processing times.
Reflecting on Model Complexity: Given the low or negative Adjusted R-squared values, it may be worth considering the removal of these interaction terms, especially if they are not theoretically justified or if the model complexity is not warranted given the data.
Reevaluation of Predictive Factors: These results may prompt a reevaluation of the included predictive factors. It suggests that other variables and model structures should be explored to find a better fit for the data. It's crucial to identify factors outside of the examined interactions that could have a more substantial impact on processing times.

The negligible Adjusted R-squared values across models underscore the minimal influence of combined centrality measures and demographic factors on the application processing times at the USPTO. This reinforces the notion that critical determinants of processing efficiency might lie outside the scope of the examined variables. The findings suggest that other, unexplored factors may offer a stronger explanation for the variability in processing times. These could include the intrinsic characteristics of patent applications, the efficiency of organizational processes, availability of resources, or the individual skill sets and expertise of the examiners. Moreover, the low explanatory power of these models hints at a substantial degree of unaccounted-for variability. This suggests a complex interaction of multiple factors affecting processing times, which warrants further investigation. Future studies might benefit from examining how the internal policies of the USPTO, the workloads assigned to individual examiners, or the intricacies inherent to various technical domains of patent applications contribute to the overall time required for processing. A more holistic approach, possibly integrating qualitative data or alternative statistical models, could yield a richer understanding and reveal opportunities for enhancing efficiency within the patent examination process.


### Refining the regression models by incorporating disposal_type and technology center (tc) variables, considering the outcome of patent applications and the specific areas of technological expertise. These additions aim to capture a more nuanced view of the factors influencing processing times, recognizing the role of content-specific knowledge and the final disposition of applications.


```{r}
# Enhancing regression models to include disposal type and technology center, alongside centrality, gender, and race interactions
results_df_2 <- map_dfr(
  centrality_measures,
  ~ run_regression(node_data,
    paste0(.x, " * gender * race + disposal_type + tc"),
    "app_proc_time",
    plot = FALSE
  )
)

# Summarizing the enhanced models by showcasing the R-squared values, offering insights into their improved explanatory capabilities
results_df_2 %>%
  select(centrality_measure, adj_r_squared) %>%
  distinct()
```

Observations:
Incorporating both disposal type and technology center categories, in addition to interactions between centrality measures and demographics, has led to a noticeable improvement in the explanatory power of the models. The Adjusted R-squared values show a substantial increase, indicating that these context-specific variables play a significant role in processing times.

Closeness Centrality's Relative Influence: With the highest Adjusted R-squared value (0.16281), the model including closeness centrality suggests that an examiner's accessibility within their professional network has the most substantial relative impact on processing times when considered with demographic and contextual factors.
Degree Centrality's Contextual Impact: The Adjusted R-squared value for degree centrality (0.12618) demonstrates that an examiner's number of direct connections is more informative about processing times when the context of their work—such as the disposal type of the application and their technological center—is taken into account.
Betweenness Centrality's Modest Role: Although the model with betweenness centrality has the lowest Adjusted R-squared value (0.12395) among the three, it still indicates that an examiner's role as a bridge within the network contributes to explaining the variance in processing times, especially when combined with additional context variables.

Implications for the USPTO:
Disposal Type and Technology Center's Significance: The fact that these factors have improved the model's fit indicates that the intricacies of the patent examination process and the examiner's area of expertise are critical to understanding processing times. These factors could be associated with the complexity of the patent subject matter or the operational dynamics within different centers.
Strategic Resource Allocation: The USPTO might use these insights for strategic planning and resource allocation, particularly by supporting examiners in roles that the model identifies as having more extended processing times.
Tailored Efficiency Improvements: Understanding the variability in processing times across different technology centers and disposal types can guide targeted efficiency improvements and training programs.
Context-Driven Network Analysis: While centrality measures alone provide limited insight, their impact becomes clearer when viewed through the lens of the specific context in which examiners operate. This suggests that future studies and network analyses should consider these contextual factors to gain a complete picture.

### To analyze the effect of disposal_type and technology center (tc) on the processing time of patent applications (app_proc_time), while considering interactions with centrality measures (degree, betweenness, and closeness), the following R code provides a structured approach combining all interaction terms. This code incorporates these variables into a linear regression model, filters for significant coefficients, and plots them to highlight the most influential factors.

```{r}
library(dplyr)
library(broom)
library(ggplot2)

# Linear regression model with interaction terms
model <- lm(app_proc_time ~ degree * tc * disposal_type + 
                             betweenness * tc * disposal_type +
                             closeness * tc * disposal_type, data = node_data)

# Summary of the model to check the overall fit and significance of predictors
summary(model)

# Tidying the model for a cleaner presentation of coefficients
tidy_model <- tidy(model)

# Filtering significant coefficients (assuming a significance level of 0.05)
significant_coeffs <- tidy_model %>% 
  filter(p.value < 0.05)

# Plotting significant coefficients
ggplot(significant_coeffs, aes(x = reorder(term, estimate), y = estimate, fill = p.value < 0.05)) +
  geom_col(show.legend = FALSE) +
  coord_flip() +
  labs(x = "Terms", y = "Estimate", title = "Significant Model Coefficients") +
  theme_minimal()

```
Significant Predictors:
(Intercept) & disposal_typeISS: The positive coefficients for the intercept and applications resulting in issued patents (disposal_typeISS) indicate that, on average, applications take a significant amount of time to process, with issued patents taking even longer. This could be due to the thorough examination required for issuing a patent.
tc1700 & tc2100: The negative coefficients for technology centers 1700 and 2100 suggest that applications in these areas tend to be processed faster than the baseline technology center. This could reflect differences in the complexity or workload associated with different technological fields.

Interaction Effects:
While many of the interaction terms are not statistically significant (p > 0.05), the model does suggest complex relationships between the centrality measures, technology centers, and disposal types. For example, the interaction between degree and disposal_typeISS has a positive coefficient, albeit not significant at traditional levels, hinting that the network position of an examiner might slightly influence processing times for issued patents.
The terms degree:tc2100:disposal_typeISS and degree:tc1700:disposal_typeISS show negative coefficients close to being significant, suggesting that for applications in technology centers 2100 and 1700 that result in issued patents, higher examiner centrality might correlate with slightly faster processing times, possibly due to more efficient handling or expertise in these areas.

Model Fit and Implications:
Multiple R-squared (0.1874): This value indicates that approximately 18.74% of the variability in application processing times is explained by the model. While this is a significant portion, it also suggests that there are other factors not captured by the model that influence processing times.
Adjusted R-squared (0.1695): The adjustment for the number of predictors in the model results in a slightly lower R-squared value, which still suggests a reasonable fit but also indicates the complexity and potential for overfitting with many predictors.
F-statistic: The F-statistic and its corresponding p-value (< 2.2e-16) indicate that the model is statistically significant overall, meaning that there is a relationship between the predictors (both individual and interaction terms) and application processing times.

Recommendations:
Focus on Significant Predictors: Given the significant predictors and interactions, USPTO might look into specific practices and policies in technology centers 1700 and 2100 to understand what might be contributing to faster processing times and whether these practices could be applied elsewhere.
Further Investigation: The interaction terms, particularly those involving degree, suggest that the network structure and examiner's position within it might play a role in processing efficiency, albeit a complex one that may vary by technology center and disposal type. Further investigation into these relationships could provide more nuanced insights.
Additional Factors: Consider incorporating additional variables into the model that could influence processing times, such as the complexity of the application, examiner workload, and applicant behavior (e.g., responsiveness to requests for information).
Model Refinement: Given the complexity of the model and the presence of many non-significant interaction terms, consider simplifying the model to focus on the most influential predictors. This might involve statistical techniques for model selection, such as stepwise regression or regularization methods, to find a balance between model complexity and explanatory power.

### To continue the analysis with a focus on data preparation and preliminary exploration, particularly addressing missing values in your dataset (presumably named applications_subset), the following R code snippets offer methods to summarize missing values across all columns and provide a detailed view of your dataset. This step is crucial before advancing with any regression analysis or model building to ensure the quality and completeness of your data.

```{r}
# Summarize the number of missing values in each column
missing_values_summary <- sapply(node_data, function(x) sum(is.na(x)))

# Print the summary
print(missing_values_summary)

```

we can see that closeness is the column with the most missing values, which may require a strategic approach to handle. Given that both closeness and gender have a substantial number of missing values, decisions on imputation or exclusion will significantly affect our analysis. Since race and tenure_days have relatively few missing values, we might opt for simpler methods. 

### Our objective is to refine the analysis of patent application processing times by integrating various predictors and examining their interactions. Before we can proceed with sophisticated modeling techniques, we need to ensure that our dataset, node_data, is clean, consistent, and suitably structured. This involves selecting relevant variables, transforming date columns into a numerical format for computational ease, and addressing any missing values that could skew our results or impede the modeling process.

As a critical step, we approach the challenge of missing data. Missing values can stem from various sources—be it data entry errors, loss of data, or variables that are sometimes unobservable. We must address this to avoid bias in our subsequent analyses. 

```{r}
# Load necessary packages
library(dplyr)
library(Amelia)

# Assign node_data to nodes and perform data preprocessing
nodes <- node_data %>%
  select(-examiner_name_middle, -abandon_date, -patent_number, -filing_date, -appl_status_date) %>%
  mutate(
    patent_issue_date = as.numeric(as.Date(patent_issue_date) - as.Date('1970-01-01')),
    earliest_date = as.numeric(as.Date(earliest_date) - as.Date('1970-01-01')),
    latest_date = as.numeric(as.Date(latest_date) - as.Date('1970-01-01'))
  )

# Copy nodes data to nodes_data and remove rows with missing app_proc_time
nodes_data <- nodes[!is.na(nodes$app_proc_time), ]

# Convert character columns to factors
character_columns <- sapply(nodes_data, is.character)
nodes_data[character_columns] <- lapply(nodes_data[character_columns], as.factor)

# Convert character variables to factors
char_cols <- sapply(nodes_data, is.character)
nodes_data[char_cols] <- lapply(nodes_data[char_cols], as.factor)

# Convert Date variables to numeric
date_cols <- sapply(nodes_data, inherits, "Date")
reference_date <- as.Date("1970-01-01")
nodes_data[date_cols] <- lapply(nodes_data[date_cols], function(x) as.numeric(difftime(x, reference_date, units = "days")))



# Define variables for imputation
impute_vars <- c("app_proc_time", "degree", "tc", "disposal_type", "betweenness", "closeness", "gender", "race")

applications_subset <- nodes_data[, impute_vars]

applications_subset$tc = as.numeric(applications_subset$tc)

nodes_data = applications_subset
# Perform multiple imputation with Amelia
amelia_output <- amelia(x = nodes_data, m = 5, noms = c("gender", "race", "disposal_type"))

# Summary of imputations
summary(amelia_output)

# Extract one of the imputed datasets (e.g., the first one)
nodes_data <- amelia_output$imputations[[1]]
```

Upon conducting a preliminary assessment of our dataset, we uncovered a considerable amount of missing data in two pivotal variables: closeness, representing the embeddedness of a patent examiner within the network (42.3% missing), and gender (14.5% missing). The absence of these data points presents us with the need to employ advanced imputation techniques to mitigate potential biases that could arise from their omission.
The variable closeness is especially crucial as it encapsulates an examiner's accessibility and influence within their professional network, which could significantly impact the processing times of patent applications. Likewise, gender may provide valuable insights into the diversity of the patent examination process, an aspect we cannot afford to overlook.
To address this, we utilized Amelia—a package that implements a multiple imputation methodology. Multiple imputation has the advantage of incorporating the inherent uncertainty of missing data into our estimates, allowing us to generate a range of plausible values that could have been observed. This process leads to more robust statistical inferences compared to single-imputation methods, which might unduly assert certainty where there is none.
Our choice to select one of the imputed datasets for analysis is driven by practical considerations, although ideally, we would pool the results across all imputed datasets to fully capture the uncertainty of our estimates. Nonetheless, even a single imputed dataset stands as a significant improvement over the incomplete one.
With the imputed values for closeness and gender, our dataset is not only more complete but also allows for a richer, more nuanced analysis that can more accurately reflect the reality of patent processing dynamics. This dataset now serves as a solid foundation for further statistical modeling, where we aim to dissect the intricate relationships and interactions between the examiners' network positions, demographic factors, and the specificities of patent applications within the broader context of technology centers and disposal type

### As we delve deeper into the dynamics influencing patent application processing times at the USPTO, we transition from traditional statistical models to more complex, non-linear machine learning techniques. Our choice of model is the random forest, a powerful ensemble learning method known for its high accuracy and robustness against overfitting.The aim of this analysis is to apply random forest, to predict patent application processing times while assessing the relative importance of various predictors including centrality measures, demographics, and other application-specific variables.

```{r}
# Load necessary libraries
library(randomForest)
library(ggplot2)

# Splitting the dataset into training and testing sets
set.seed(123) # For reproducibility
training_indices <- sample(1:nrow(nodes_data), 0.7 * nrow(nodes_data))
train_data <- nodes_data[training_indices, ]
test_data <- nodes_data[-training_indices, ]

# Training the Random Forest model on the training set
rf_model <- randomForest(app_proc_time ~ degree + tc + disposal_type + betweenness + closeness + gender + race,
                         data = train_data,
                         ntree = 300,
                         importance = TRUE)

# Predicting on the testing set
predictions <- predict(rf_model, newdata = test_data)

# Accessing the R-squared value directly from the model
rsq <- rf_model$rsq
cat("R-squared value from the model: ", tail(rsq, 1), "\n") # Display the last (most relevant) R-squared value

# Calculate Mean Absolute Percentage Error
mape <- mean(abs((train_data$app_proc_time - predictions) / train_data$app_proc_time)) * 100

# Print the MAPE
cat("Mean Absolute Percentage Error: ", as.character(mape), "\n")

# Assuming "%IncMSE" is available and relevant for feature importance
importance_type <- if ("%IncMSE" %in% colnames(importance(rf_model))) "%IncMSE" else "IncNodePurity"

# Extracting feature importance using the chosen importance type
featureImportance <- data.frame(Feature = rownames(importance(rf_model)), 
                                Importance = importance(rf_model)[, importance_type])


# Plotting feature importance for better visualization
feature_importanc <- ggplot(featureImportance, aes(x = reorder(Feature, Importance), y = Importance)) +
  geom_bar(stat = 'identity', fill = 'steelblue') +
  coord_flip() +
  labs(x = "Features", y = "Importance", title = "Feature Importance in Random Forest") +
  theme_minimal()

feature_importanc
```
The R-squared value is approximately 0.111, which suggests that about 11.1% of the variance in application processing times can be explained by the model. This is a relatively low value, indicating that while the model does capture some of the patterns in the data, there is a significant amount of variability that is not accounted for by the included variables.The MAPE is about 74.02%, which indicates that the predictions of the model are on average 74.02% off from the actual processing times. This suggests that the model's predictive accuracy is not very high, and there could be substantial deviations between the predicted and actual values.

disposal_type is the most important feature, with the highest score, suggesting it has the strongest relationship with application processing times within the model.
The tc (technology center) is the next most important feature, indicating that the specific technology center handling the application is also a significant predictor of processing times.
degree follows in importance, showing that the network centrality measure has a notable but lesser influence than disposal_type or tc.
closeness and betweenness have moderate importance scores, suggesting they contribute to the model's predictions but are less influential than the top features.
race and gender have the lowest importance scores, suggesting that these demographic factors have a relatively small impact on application processing times in the context of the other variables in the model.

Implications:

The dominance of disposal_type in feature importance may indicate that the outcome of the patent application (e.g., approved, rejected, withdrawn) is closely linked to how long the processing takes.
The technology center's impact could reflect differences in complexity, specialization, or workflow efficiency among the different centers.
Lower importance of race and gender could imply that these factors do not play a major role in processing times, or that any effect they do have is overshadowed by the other factors in the model.
The model's overall low explanatory power and high MAPE suggest that there may be other unmodeled factors that significantly influence processing times, or that the relationships between the predictors and the processing times are highly complex and not easily captured by the model.

### The objective here is to explore the efficacy of a Support Vector Regression (SVR) model, specifically utilizing a linear kernel, in predicting the processing times for patent applications. The SVR model is employed to discern linear associations between various features of the application process and the resulting processing times. The analysis entails fitting the SVR model on a subset of the data designated for training and subsequently gauging its predictive accuracy on a separate testing set. By calculating metrics like the R-squared value and the Mean Absolute Percentage Error (MAPE), we aim to quantify the model's predictive performance. Further, the examination of residuals through density plots will shed light on the distribution of the prediction errors, thereby offering insights into potential model improvements for future iterations.

```{r}
# Load necessary libraries
library(e1071)
library(ggplot2)

# Ensure your SVM model uses a linear kernel for this approach
svm_model <- svm(app_proc_time ~ ., data = train_data, kernel = "linear")

# Print the summary of the SVM model to check the model details
print(summary(svm_model))

# Predicting on the testing set
predictions <- predict(svm_model, newdata = test_data)
observed <- test_data$app_proc_time

# Calculate R-squared value
rss <- sum((predictions - observed)^2)
tss <- sum((observed - mean(observed))^2)
rsquared <- 1 - rss / tss
cat("R-square value is ", as.character(rsquared), "\n")

# Calculate Mean Absolute Percentage Error (MAPE)
mape <- mean(abs((observed - predictions) / observed)) * 100
cat("Mean Absolute Percentage Error: ", mape, "\n")

# Calculate and plot residuals
residuals <- predictions - observed

# Density plot of residuals
plot(density(residuals), main = "Density of Residuals", xlab = "Residuals")
abline(v = 0, col = "red")  # Vertical line at 0

```
The density plot of the residuals shows the distribution of the differences between the predicted and actual processing times. The vertical red line represents the point of zero residuals, where a prediction perfectly matches the actual value. The plot suggests that the residuals are centered around zero and have a bell-shaped distribution, which is generally a good sign indicating that the errors are normally distributed. However, there are tails on both ends, suggesting some predictions are significantly off from the true values. A skew or heavier tails could indicate that the model struggles with certain types of applications.

The R-squared value of about 0.077 suggests that the model explains approximately 7.7% of the variability in the processing times. This is relatively low, implying that the linear model does not capture much of the complexity or variance in the processing times. It indicates limited predictive power within the context of the data provided to the model.The MAPE of approximately 67.75% indicates that the model's predictions are on average about 67.75% off from the actual processing time. This is a substantial error rate, which signals that the predictions are not very accurate on a relative scale.

The results suggest that the linear SVR model has limited predictive accuracy in this context. While the residuals are approximately normally distributed, the low R-squared value and high MAPE signal that the model's predictions are often far from actual processing times. This could be due to various factors such as non-linear relationships in the data that are not captured by a linear kernel, or important predictors missing from the model. There may be a need to explore more complex models or feature engineering to improve the predictive performance.


### Looking forward to explore Gradient Boosting Machine (GBM) model to predict patent application processing times and to evaluate the model's performance. We aim to determine the most influential factors affecting processing times by analyzing feature importance and assess the accuracy of the model using statistical metrics such as R-squared and MAPE. The residuals from the model's predictions will also be examined to ensure the model's reliability.

```{r, fig.keep = "last"}
# Load necessary libraries
library(gbm)
library(ggplot2)

# Set seed for reproducibility
set.seed(123)

# Assuming nodes_data is your dataset and app_proc_time is your target variable
# Set parameters and train the GBM model
gbm_model <- gbm(app_proc_time ~ ., 
                 data = nodes_data, 
                 distribution = "gaussian",
                 n.trees = 500,
                 interaction.depth = 3,
                 shrinkage = 0.01,
                 cv.folds = 5,
                 n.minobsinnode = 10)

# Perform cross-validation to determine the optimal number of trees
best_trees <- gbm.perf(gbm_model, method = "cv")
cat("Best number of trees: ", best_trees, "\n")

# Predictions for the observed data
predictions <- predict(gbm_model, n.trees = best_trees)

# Calculate R-squared value
rsquared <- cor(predictions, nodes_data$app_proc_time)^2
cat("R-squared value: ", rsquared, "\n")

# Calculate Mean Absolute Percentage Error (MAPE)
mape <- mean(abs((nodes_data$app_proc_time - predictions) / nodes_data$app_proc_time)) * 100
cat("Mean Absolute Percentage Error: ", mape, "\n")

# Calculate residuals
residuals <- nodes_data$app_proc_time - predictions

# Density plot of residuals
residual_plot <- plot(density(residuals), main = "Density of Residuals", xlab = "Residuals")
abline(v = 0, col = "red")  # Vertical line at 0

```

```{r}
# Calculate feature importance and transform it for ggplot2
feature_importance <- summary(gbm_model, n.trees = best_trees)
feature_importance_df <- data.frame(
  Feature = rownames(feature_importance),
  Importance = feature_importance$rel.inf
)

# Plot feature importance with ggplot2
feature_importance_plot <- ggplot(feature_importance_df, aes(x = reorder(Feature, Importance), y = Importance)) +
  geom_bar(stat = "identity") +
  coord_flip() +  # Flip coordinates to make the plot horizontal
  xlab("Features") +
  ylab("Relative Importance") +
  ggtitle("Feature Importance Plot")
plot(feature_importance_plot)
```

## Discard the first feature importance plot, it is being automatically generated. USe one in black/grey

The first plot likely represents the cross-validation error as a function of the number of trees in the GBM model. The error rapidly decreases and then plateaus, indicating the model improves with more trees initially but gains little after a certain point. The vertical dashed line may indicate the optimal number of trees (444) determined through cross-validation, which minimizes the error.

The feature importance plot visualizes the relative influence of each predictor variable on the model. disposal_type is the most influential feature, followed by tc (technology center). This suggests that the type of disposal and the specific technology center associated with the application are critical in predicting processing times. Other variables like degree, closeness, betweenness, race, and gender have less relative importance, with gender being the least influential among them.

The density of residuals plot shows the distribution of the differences between the observed and predicted processing times. Ideally, this should be centered around zero and normally distributed, indicating good model fit. In this case, the distribution is approximately centered around zero but appears to be slightly right-skewed, suggesting that the model may be underpredicting some of the processing times.

An R-squared value of 0.169 suggests that approximately 16.9% of the variance in the patent application processing times is explained by the model. While not high, it indicates that the model has captured a portion of the relationship between the predictors and the response variable.A MAPE of approximately 67.75% indicates that, on average, the model's predictions are about 67.75% off from the actual values. This relatively high value suggests that the predictions can vary widely from the actual processing times and that there may be room for improvement in the model.

The GBM model has provided some insights into the factors that affect patent application processing times, but the predictive accuracy has significant room for improvement. The model's R-squared and MAPE indicate that while some trends have been identified, the model may not be capturing all the complexities of the data. The underrepresented predictive accuracy suggests further model tuning or the inclusion of additional relevant features could be beneficial. The results may also imply that the relationships in the data are not purely linear or additive, and interactions or non-linear terms could be significant.

### The intent of this analysis is to deploy an XGBoost regression model to ascertain the determinants of patent application processing times. By leveraging a robust ensemble learning approach, we seek to capture complex nonlinear relationships and interactions between predictors, with the goal of improving the accuracy and reliability of our predictions.The XGBoost algorithm, known for its performance in both classification and regression tasks, operates by constructing a series of decision trees in a gradient boosting framework. This methodology allows for incremental learning from the data, where each subsequent tree attempts to correct the errors of its predecessors.

```{r}
# Load the xgboost package for modeling and Metrics for evaluation
library(xgboost)
library(Metrics)

# Set a seed for reproducibility of results
set.seed(123)

# Ensure that all factor columns in the dataset are converted to numeric
# We can achieve this by coercing factors to integers, as xgboost can handle integer encoding for categorical data
numeric_train_data <- data.frame(lapply(train_data, function(x) if(is.factor(x)) as.integer(x) else x))
numeric_test_data <- data.frame(lapply(test_data, function(x) if(is.factor(x)) as.integer(x) else x))

# Create xgb.DMatrix objects for the training and test sets
dtrain <- xgb.DMatrix(data = as.matrix(numeric_train_data[, -which(names(numeric_train_data) == "app_proc_time")]),
                      label = numeric_train_data$app_proc_time)
dtest <- xgb.DMatrix(data = as.matrix(numeric_test_data[, -which(names(numeric_test_data) == "app_proc_time")]),
                     label = numeric_test_data$app_proc_time)

# Define the parameters for the xgboost model
params <- list(
  booster = "gbtree",
  objective = "reg:squarederror",
  eta = 0.01,
  max_depth = 3,
  subsample = 0.7,
  colsample_bytree = 0.7
)

# Specify the number of boosting iterations
nrounds <- 500

# Train the xgboost model
xgb_model <- xgboost(params = params, data = dtrain, nrounds = nrounds, nthread = 1, verbose = 0)

# Predict the app_proc_time using the model for the test set
predictions <- predict(xgb_model, dtest)

# Compute the R-squared value
rsquared <- cor(predictions, numeric_test_data$app_proc_time)^2
cat("R-squared value: ", rsquared, "\n")

# Calculate the Mean Absolute Percentage Error (MAPE)
mape <- mean(abs((numeric_test_data$app_proc_time - predictions) / numeric_test_data$app_proc_time)) * 100
cat("Mean Absolute Percentage Error: ", mape, "\n")

# Feature importance analysis
# Extract feature importance scores and plot them
importance_matrix <- xgb.importance(model = xgb_model)
importance_plot <- xgb.plot.importance(importance_matrix)
print(importance_plot) # Explicitly print if needed

# Residual analysis
# Calculate residuals as the difference between observed and predicted values
residuals <- numeric_test_data$app_proc_time - predictions

# Plot the density of residuals to assess the distribution of prediction errors
residual_plot <- plot(density(residuals), main = "Density of Residuals", xlab = "Residuals")
abline(v = 0, col = "red") # Add a vertical line at 0 for reference
print(residual_plot) # This line is not strictly necessary for base R plots, but included for consistency
```
The plot suggests that disposal_type is the most significant predictor of patent application processing times, followed by closeness and tc (technology center). Variables like degree, betweenness, race, and gender appear to have a lesser impact. This information is crucial for understanding which factors most influence the length of time a patent application will take to process.

An R-squared value of approximately 0.128 indicates that the XGBoost model explains about 12.8% of the variance in patent application processing times. This value, while not particularly high, suggests the model has captured a meaningful portion of the data's structure, but there's still much unexplained variance, which may be due to other factors not included in the model or inherent noise in the data. The MAPE of approximately 73% is relatively high, which means that the model's predictions are on average 73% off from the actual processing times. A high MAPE value suggests that the model's predictions may not be very precise, and there could be a significant margin of error in the estimated processing times.

While the model has identified some key features as important, the overall predictive power remains moderate, and the precision of the predictions could benefit from improvement. The most substantial variable, disposal_type, might reflect the outcome's impact on the time required to process applications, whereas the importance of closeness could point to the effect of an examiner's network position on their efficiency. The results indicate potential areas for further exploration and refinement, such as incorporating additional explanatory variables, considering different modeling techniques, or diving deeper into feature engineering to improve the model's performance.

--------------------------------------------------------------------------------------------------

In the pursuit to enhance the efficiency of patent processing at the USPTO, we embarked on a comprehensive analytical journey, utilizing a spectrum of statistical techniques and machine learning models to unearth the determinants of processing times. This analysis aimed to pinpoint variables that significantly affect the duration of patent examination and provide actionable insights to streamline the patent application process.

Key Findings and Observations:

Centrality in the Examiner Network: We observed that centrality measures, particularly closeness centrality, had a tangible impact on processing times. Examiners who are central to the USPTO's network are pivotal in information dissemination, potentially affecting efficiency. However, the predictive strength of these centrality measures varied, with closeness centrality being a more prominent factor than betweenness or degree centrality. This suggests a more nuanced relationship between an examiner's network position and their performance, hinting at the complex social dynamics within the USPTO.
Demographic Considerations: Interestingly, demographic factors such as race and gender, while important from an organizational and ethical standpoint, did not significantly influence processing times in the models. This finding may indicate that the USPTO's examination process is driven more by professional roles and responsibilities rather than the demographic characteristics of the examiners.
Organizational Structures' Influence: The disposal type of the application and the specific technology center (tc) emerged as critical factors with a substantial effect on processing times. These factors reflect the organizational and operational aspects of the patent examination process, suggesting that certain technology centers might be more efficient, or that some disposal types inherently take longer to process.
Model Performance: Across the various models implemented, none achieved an exceptionally high R-squared value, indicating room for improvement in capturing the complexities of the USPTO's processes. High MAPE values further suggested that the actual processing times could differ significantly from the models' predictions, highlighting the potential for enhancing model accuracy.
Residual Patterns: Residual analysis pointed towards potential systematic errors in predictions, such as underprediction of processing times for certain applications. This suggests that our models may not fully capture all influential factors, particularly those that cause deviations in processing times.

Strategic Recommendations for USPTO:
Broader Variable Integration: To capture the complex realities of the patent examination process, future models should incorporate more granular data on patent complexity, examiner expertise, and workload distribution.
Resource Optimization: Insights from technology center and disposal type variables can guide the USPTO in resource allocation, helping to optimize workflows and potentially distribute workloads more evenly among examiners.
Model Refinement: Given the modest performance of the models, there is a need to refine them further. This could include exploring non-linear models, interaction terms, or advanced machine learning algorithms that can handle the multifaceted relationships inherent in the data.

Model Recommendations
Given the detailed analysis, the GBM model stands out as the most suitable for the USPTO. It achieves a balance between capturing complex interactions within the data and offering actionable insights through feature importance. The GBM model has demonstrated its capability to discern the nuanced effects of variables on patent processing times and can be fine-tuned further for enhanced accuracy.
It is advisable for the USPTO to leverage the GBM model for continued analysis, using its results to identify areas for process optimization. The interpretability of the GBM model ensures that insights can be effectively communicated to stakeholders, making it a valuable tool for informed policy-making and strategic planning. Further, the model’s flexibility to integrate new data as it becomes available makes it a robust choice for an evolving organizational environment like the USPTO.


Suggestions for improving accuracy (improving Rsquare and deceasing MAPE)
To enhance the accuracy  of analysis for predicting patent application processing times at the USPTO, several targeted strategies can be adopted. These include refining feature engineering by integrating more detailed temporal data and creating new features from existing data, such as interactions between demographics and technology centers. The potential use of text analysis on examiner comments and patent content can further reveal complexities influencing processing times. Addressing demographic impacts, advanced statistical models like mixed-effects models can accommodate the hierarchical data structure, and boosting algorithms can effectively handle complex interactions.

Improving data quality through rigorous cleaning and integrating additional sources will enrich the dataset, providing a more robust foundation for analysis. Employing stratified sampling for model validation ensures that the training and testing sets represent the diversity of the dataset, especially important for demographic analyses. Finally, enhancing model interpretation and conducting sensitivity analyses will provide clearer insights into the factors affecting processing times, facilitating more informed decision-making and highlighting areas for organizational improvement at the USPTO.