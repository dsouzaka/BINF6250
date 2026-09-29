# Introduction
In this project, the aim was to learn about how to program Markov models through generating literary texts based on input texts. A Markov chain predicts the next word based on how often each word followed the previous word or words in the training text. We had  worked with a Seuss text line for creating a 1st order Markov chain. Then we scaled up the function to accommodate any N order Markov model. After that, we had worked on two other functions to choose the next probable state (next word in this case), and to append the states to create a run-on sentence string of the final text. Finally, we had put all three functions together using the input of all of Shakespeare's sonnets to generate a new sonnet.

# Pseudocode

## 1st-order Markov model
```
FUNCTION build_markov_model(markov_model, new_text):

    # Step 1: Split text into a list of words
    words = split new_text on whitespace

    # Step 2: Add artificial start and end states
    words = ['*S*'] + words + ['*E*']

    # Step 3: Walk through consecutive word pairs
    FOR i FROM 0 TO length(words) - 2:
        current_state = words[i]
        next_state = words[i + 1]

        # Step 4: Ensure current_state exists as a key in markov_model
        IF current_state NOT IN markov_model:
            markov_model[current_state] = empty dictionary

        # Step 5: Ensure next_state exists as a key in the inner dictionary
        IF next_state NOT IN markov_model[current_state]:
            markov_model[current_state][next_state] = 0

        # Step 6: Increment the transition count
        markov_model[current_state][next_state] += 1

    # Step 7: Return the updated model
    RETURN markov_model
```

## Nth-order Markov model
```
FUNCTION build_markov_model(markov_model, text, order):

    # Step 1: If no model was given, start a new one
    IF markov_model is None:
        markov_model = empty dictionary

    # Step 2: Split text into a list of words
    words = split text on whitespace

    # Step 3: Add `order` copies of '*S*' to the start and one '*E*' to the end
    words = ['*S*'] * order + words + ['*E*']

    # Step 4: Walk through the text using a sliding window of size order + 1
    FOR i FROM 0 TO length(words) - order - 1:
        current_state = tuple(words[i : i + order])
        next_state = words[i + order]

        # Step 5: Ensure current_state exists as a key in markov_model
        IF current_state NOT IN markov_model:
            markov_model[current_state] = empty dictionary

        # Step 6: Ensure next_state exists as a key in the inner dictionary
        IF next_state NOT IN markov_model[current_state]:
            markov_model[current_state][next_state] = 0

        # Step 7: Increment the transition count
        markov_model[current_state][next_state] += 1

    # Step 8: Return the updated model
    RETURN markov_model
```

## Get next word
```
FUNCTION get_next_word(current_word, markov_model, seed):

    # Step 1: Look up the possible next words and their counts
    possible_words = markov_model[current_word]

    # Step 2: Calculate the total count of all possible next words
    total_count = sum of possible_words' values

    # Step 3: Calculate the transition probability for each next word
    FOR each word, count IN possible_words:
        probabilities[word] = count / total_count

    # Step 4: Extract the words and probabilities as two aligned lists
    words_list = list of probabilities' keys
    probs_list = list of probabilities' values

    # Step 5: Randomly select the next word, weighted by its probability
    # (seeding happens once in generate_random_text, not here)
    next_word = weighted random choice from words_list using probs_list

    # Step 6: Return the selected next word
    RETURN next_word
```

## Generate random text
```
FUNCTION generate_random_text(markov_model, seed):

    # Step 1: Determine the model's order from the length of any key
    some_key = any key from markov_model
    order = length of some_key

    # Step 2: Initialize the current state with `order` copies of '*S*'
    current_word = tuple of `order` copies of '*S*'

    # Step 3: Set the random seed once, then get the first word
    set random seed to seed
    next_word = get_next_word(current_word, markov_model)

    # Step 4: Repeat until the end marker is generated or a safety limit is reached
    sentence = empty list
    count = 0
    WHILE next_word is not '*E*':

        # Step 5: Add the generated word to the sentence
        APPEND next_word to sentence

        # Step 6: Slide the window forward using the new word
        current_word = current_word[1:] + (next_word,)

        # Step 7: Get the next word
        next_word = get_next_word(current_word, markov_model)

        # Step 8: Stop early if generation runs unexpectedly long
        count += 1
        IF count > 2000:
            BREAK

    # Step 9: Join the generated words into a single space-separated string
    sentence = join words with spaces

    # Step 10: Return the final sentence
    RETURN sentence
```

# Successes
We were able to collaborate across two online meetings via screen sharing, and did a good job of improvising ideas in pseudocode for the code before resorting to Google or AI resources. Working through each code chunk sequentially helped us talk through the purpose of each function and which data structure works best for each scenario, whether it be a for loop, a tuple, a dictionary, etc. We had learned how Markov chains are coded, and how to use probabilities and random number generators to determine the next state.

# Struggles
One of the struggles our team faced occurred when writing the `get_next_word()` function. Our initial plan was to have the function calculate all the probabilities, and then find the highest probability. It was originally going to choose the word with the highest probability as the next word, and randomize it if it was a tie. This became a problem when running the `generate_random_text()` function because certain phrases (Black fish, Blue fish, Black fish, Blue fish...) were getting stuck in a repeating loop since the same word was picked every time. We ultimately changed the function to select the next word randomly, weighted by the probabilities, rather than always choosing the most likely next word. This gave lower probability words a chance of being selected, which fixed the looping caused by always picking the most likely word.

We also ran into two seeding issues. At first, `generate_random_text()` wasn't passing the seed to `get_next_word()`, so every seed had the same output. We fixed that, but then peer review helped us realize that setting the seed inside `get_next_word()` reset it on every call, so each step used the same random draw. This made the text loop again, and the 2000-word safety limit hid this. We ultimately fixed this by setting the seed once at the start of `generate_random_text()`.

# Personal Reflections
## Group Leader
Katelyn - I found this project to be much more involved than the first one. I have some experience with Markov chains from taking genomics last semester; however, creating the algorithms required to build a Markov chain from a text and then produce a new text using the model was definitely new to me. I initially struggled a bit with understanding where to start writing these functions, but having multiple calls with my group to discuss approaches and write the pseudocode definitely clarified the objective of this project. As this was my first time being group leader, it did take me a bit of time to understand how to merge and commit different pull requests, and I definitely had a hard time with it when our group was working on the same file at the same time; my team member created a branch and pull request, and then I made edits directly to the README file, so when I merged her work, my edits got overwritten. This project helped me get a better understanding of how GitHub works from the project leader's end. It also helped me better understand the nuances between data structures, like how using tuples as dictionary keys let us build higher-order Markov models, but we had to consider mutability differences between tuples and lists. Overall, I believe that our group was able to work together and complete the project to the best of our abilities.

## Other members
Abby - I found this project to be very interesting and informative. I have no experience with these models, and especially after being sick during the lecture, I had a lot of ground to cover to feel okay working on these functions. I found the structures used for Markov models to be very interesting, with dicts of dicts, etc. It was difficult for me to understand the key-value pairs and lists needed for the generation of random text in the beginning, but after looking at what the function inputs needed to be, it made more sense. I was also confused by the seed=42 for a little while and had to ask Claude why 42 in order to understand that the random number gen seed just ensures the ability to reproduce data. 

Yulia - I found this project to be an important learning moment, as I had no awareness of how Markov chains worked prior to the last class. Through working the code possibilities with my group members, we had learned how to program current and next states for any N order of Markov model, and then use the random number generator to pick the next probable state. I had learned about the p= segment of the numpy random number generator function, which applies probability bias when picking the next state, where if it is higher probability it is more likely that word would be chosen, and if the probabilities were the same then it would be randomly picked. This worked much better than the initial loop we had in our original code, where it would select the maximum probability if there was one, and if there is not one, then pick the next state randomly. However this approach created an issue of it being stuck in a loop. It was also interesting to learn about how to store the key value pairs with tuples and dictionaries. 

# Generative AI Appendix
Claude was used to identify the use of the seed in the generate and next word functions. I asked, 'In the context of a Markov model, what is the function of a seed, and does the number have any significance?' I learned that it is just to maintain randomness while keeping the ability to reproduce results.
Claude was also used to help clean up the structure of the pseudocode that we had brainstormed on call. I asked, 'Would you be able to restructure this pseudocode to be step-by-step and clean up any grammar mistakes?' and Claude was able to help me quickly reformat it to ensure readability while still keeping our initial plans.
