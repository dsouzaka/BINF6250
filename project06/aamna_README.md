A description of the project
Pertinent documentation and usage write-up
Pseudocode

```
def build_distance_matrix()

Get all sequence IDs from the dictionary and count them (n)

Create an n × n matrix filled with zeros
    Note: The diagonal stays zero since a sequence has no distance from itself.

FOR each sequence i:
    FOR each sequence j starting at i + 1:
        Note: No need to compare every pair twice since distance(A, B) = distance(B, A).
        Starting at i + 1 means we only calculate the upper triangle of the matrix.

        Get seq1 and seq2 using their IDs

        similarity = smith_waterman(seq1, seq2)
        Note: this is where we build on the alignment function from Project 05 instead of starting over.

        distance = 1 - similarity
        Note: We're building a distance matrix, not a similarity matrix. More similar sequences should have smaller distances.

        Store distance in matrix[i][j] and matrix[j][i]
        Note: Filling both positions keeps the matrix symmetric.

RETURN distance_matrix, ids
Note: Keep the IDs in order because the matrix itself doesn't tell us which sequence is which.


def neighbor_joining()
Calculate a corrected distance matrix
Find the pair with the smallest corrected distance.
Create a new internal node joining those two sequences.
Calculate the branch lengths connecting them.
Replace the two sequences with their new internal node.
Update the distance matrix.
Repeat until only two nodes remain, then connect them.

IF only 2 nodes remain:
    final_distance = distance_matrix[0][1] / 2
    RETURN the two nodes connected in Newick format

    Note: Chose 2 nodes as the stopping condition and split the final distance equally to preserve the distance between them.

n = number of labels
row_sums = sum of each row in the distance matrix

Set min_score = infinity
Set min_i and min_j = 0

FOR each node i:
    FOR each node j starting at i + 1:

        corrected_distance = (n - 2) × distance_matrix[i][j] - row_sums[i] - row_sums[j]

        IF corrected_distance < min_score:
            Update min_score, min_i, and min_j

    Note: NJ uses corrected distances (D*) rather than just joining the pair with the smallest raw distance.

distance_ij = distance_matrix[min_i][min_j]

branch_i = distance_ij / 2 + (row_sums[min_i] - row_sums[min_j]) / (2 × (n - 2))
branch_j = distance_ij - branch_i

Create new_node = (label_i:branch_i, label_j:branch_j)
    Note: Building the Newick string as we go is quicker not needing a separate function to construct the tree.

Keep all indices except min_i and min_j

Create a new (n - 1) × (n - 1) distance matrix
Copy the distances between the remaining nodes

FOR each remaining node k:
    new_distance = (distance_matrix[min_i][k] + distance_matrix[min_j][k] - distance_ij) / 2
    Store new_distance in the last row and column

Create new_labels using the remaining labels + new_node

RETURN neighbor_joining(new_distances, new_labels)

```
Reflection narrative that describes what went right, what went wrong, what you didn't understand, how it could have been better, and how you could have been better.
Appendix (AI usage, citations, etc.)
