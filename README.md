 
import os
import random
from PIL import Image

# --- UTILITY: Placeholder Image Generation ---

def create_placeholder_image(filename="input_image.png", size=(128, 128)):
    """Creates a visually rich 128x128 grayscale diagonal gradient image if one is not found."""
    if os.path.exists(filename):
        return

    print(f"Placeholder image '{filename}' not found. Generating a default diagonal gradient image...")
    
    # Create a new grayscale image
    img = Image.new('L', size)
    pixels = img.load()
    width, height = img.size
    
    # Generate a diagonal gradient (0 to 255 across the image)
    max_sum = width + height - 2 # Maximum value of x + y
    
    for y in range(height):
        for x in range(width):
            # Calculate the grayscale value based on the position (x+y), normalized to 255
            # This creates a smooth diagonal transition from black (top-left) to white (bottom-right)
            grayscale_value = int(((x + y) / max_sum) * 255)
            pixels[x, y] = grayscale_value
                
    img.save(filename)
    print(f"Generated placeholder image: {filename} ({width}x{height} Diagonal Gradient). Simulation can now proceed.")

# --- I. Hamming (7,4) Code Implementation ---

def _bit_list_to_int(bits):
    """Converts a list of 1s and 0s (bits) to an integer."""
    if not bits:
        return 0
    return int("".join(map(str, bits)), 2)

def _int_to_bit_list(n, length=4):
    """Converts an integer to a list of bits, padded to 'length'."""
    # Format to a binary string, strip '0b', pad with leading zeros, convert to list of integers
    binary_string = format(n, '0{}b'.format(length))
    return [int(b) for b in binary_string]

def hamming_encode_4_to_7(data_bits):
    """
    Encodes 4 data bits (d1, d2, d3, d4) into a 7-bit Hamming code.
    Output format: [p1, p2, d1, p3, d2, d3, d4]
    """
    d1, d2, d3, d4 = data_bits

    # Calculate parity bits (using XOR)
    p1 = d1 ^ d2 ^ d4
    p2 = d1 ^ d3 ^ d4
    p3 = d2 ^ d3 ^ d4

    # The 7-bit codeword: [p1, p2, d1, p3, d2, d3, d4]
    codeword = [p1, p2, d1, p3, d2, d3, d4]
    return codeword

def hamming_decode_7_to_4(codeword, apply_correction=True):
    """
    Decodes a 7-bit Hamming codeword, extracts data, and optionally corrects a single error.
    Returns (data_bits, was_corrected).
    """
    # Create a copy so we don't modify the original list if we are just checking for uncorrected output
    codeword_copy = list(codeword)
    
    # Unpack codeword: [c1, c2, c3, c4, c5, c6, c7] -> [p1, p2, d1, p3, d2, d3, d4]
    c1, c2, c3, c4, c5, c6, c7 = codeword_copy

    # Recalculate parity bits (S1, S2, S3) based on received data
    s1 = c1 ^ c3 ^ c5 ^ c7
    s2 = c2 ^ c3 ^ c6 ^ c7
    s3 = c4 ^ c5 ^ c6 ^ c7

    # Calculate the syndrome (error position)
    syndrome = _bit_list_to_int([s3, s2, s1])
    was_corrected = False

    if syndrome != 0 and apply_correction:
        # Single error detected and correction is enabled. Flip the error bit.
        error_index = syndrome - 1
        codeword_copy[error_index] = 1 - codeword_copy[error_index]
        print(f"Correction applied! Flipped bit at position {syndrome} in a block.")
        was_corrected = True

    # Extract the data bits from the (potentially corrected or uncorrected) codeword
    # Indices in the 0-based codeword: 2, 4, 5, 6
    data_bits = [codeword_copy[2], codeword_copy[4], codeword_copy[5], codeword_copy[6]]

    return data_bits, was_corrected

# --- II. Channel Simulation ---

def simulate_noisy_channel(encoded_bits, error_rate=0.01):
    """
    Simulates a noisy channel by randomly flipping bits based on the error_rate.
    """
    print(f"\n--- Simulating Noisy Channel (Error Rate: {error_rate * 100}%) ---")
    noisy_bits = []
    total_errors_introduced = 0
    errors_in_blocks = 0

    # Group bits into 7-bit blocks to track multi-bit errors
    for i in range(0, len(encoded_bits), 7):
        block = encoded_bits[i:i+7]
        errors_in_block = 0
        noisy_block = []

        for bit in block:
            if random.random() < error_rate:
                # Flip the bit
                noisy_bit = 1 - bit
                total_errors_introduced += 1
                errors_in_block += 1
            else:
                noisy_bit = bit

            noisy_block.append(noisy_bit)

        noisy_bits.extend(noisy_block)
        if errors_in_block > 1:
            # If a block has more than 1 error, the Hamming code will fail
            errors_in_blocks += 1

    print(f"Total encoded bits: {len(encoded_bits)}")
    print(f"Total errors introduced: {total_errors_introduced}")
    print(f"Blocks with >1 error (Hamming correction will FAIL): {errors_in_blocks}")
    print("----------------------------------------------------------")
    return noisy_bits

# --- III. Image Processing Pipeline ---

def decode_and_reconstruct(noisy_encoded_bits, width, height, output_path, apply_correction):
    """Decodes the noisy bits and reconstructs the image, optionally applying correction."""
    received_bytes = []
    
    # The loop must advance by 14 bits because two 7-bit blocks make up one byte
    for i in range(0, len(noisy_encoded_bits), 14):
        # Process 7-bit block 1 (High Nibble)
        block1 = noisy_encoded_bits[i:i+7]
        corrected_bits1, _ = hamming_decode_7_to_4(block1, apply_correction)

        # Process 7-bit block 2 (Low Nibble)
        block2 = noisy_encoded_bits[i+7:i+14]
        corrected_bits2, _ = hamming_decode_7_to_4(block2, apply_correction)

        # Reassemble the two 4-bit nibbles back into one 8-bit byte
        byte_int = (_bit_list_to_int(corrected_bits1) << 4) | _bit_list_to_int(corrected_bits2)
        received_bytes.append(byte_int)

    # Create new image
    received_data = bytes(received_bytes)
    received_img = Image.frombytes('L', (width, height), received_data)

    # Save the resulting image
    received_img.save(output_path)
    if apply_correction:
        print(f"Successfully saved the RECEIVED AND CORRECTED image to: {output_path}")
    else:
        print(f"Successfully saved the RECEIVED BUT UNCORRECTED image to: {output_path}")


def process_image(input_path, output_path_corrected, output_path_uncorrected, error_rate):
    """
    Main function to load an image, encode, transmit, decode, and save the results.
    """
    try:
        # 1. Load Image and prepare data
        img = Image.open(input_path).convert('L') # Convert to Grayscale (single channel)
        width, height = img.size
        print(f"Loaded image: {input_path} ({width}x{height} Grayscale)")

        # Get the raw byte data (pixel values 0-255)
        raw_bytes = list(img.tobytes())

        # 2. Encode Phase: Bytes -> Bits -> 4-bit chunks -> 7-bit codewords
        encoded_bits = []
        for byte_value in raw_bytes:
            # Each byte (8 bits) is split into two 4-bit nibbles
            nibble1 = byte_value >> 4    # High 4 bits
            nibble2 = byte_value & 0b1111 # Low 4 bits

            bits1 = _int_to_bit_list(nibble1, 4)
            bits2 = _int_to_bit_list(nibble2, 4)

            # Encode both nibbles
            encoded_bits.extend(hamming_encode_4_to_7(bits1))
            encoded_bits.extend(hamming_encode_4_to_7(bits2))

        # 3. Transmission Phase: Introduce noise
        noisy_encoded_bits = simulate_noisy_channel(encoded_bits, error_rate)
        
        # 4. Decode Phase 1: Apply correction
        print("\n--- Decoding and Correcting Errors ---")
        decode_and_reconstruct(noisy_encoded_bits, width, height, output_path_corrected, apply_correction=True)

        # 4. Decode Phase 2: Show raw corrupted output (no correction applied)
        print("\n--- Decoding WITHOUT Correction (To show the noise) ---")
        decode_and_reconstruct(noisy_encoded_bits, width, height, output_path_uncorrected, apply_correction=False)

    except FileNotFoundError:
        print(f"Critical Error: Input file not found at {input_path}.")
    except Exception as e:
        print(f"An unexpected error occurred: {e}")

# --- IV. Execution ---

if __name__ == "__main__":
    INPUT_FILE = "input_image.png"
    OUTPUT_FILE_CORRECTED = "output_corrected_image.png"
    OUTPUT_FILE_UNCORRECTED = "output_uncorrected_image.png"
    
    # Error rate set to 5% to clearly demonstrate the failure rate of the Hamming code
    CHANNEL_ERROR_RATE = 0.05

    # 0. Check for input file and create placeholder if missing
    create_placeholder_image(INPUT_FILE)

    # 1. Run the main process
    if os.path.exists(INPUT_FILE):
        process_image(INPUT_FILE, OUTPUT_FILE_CORRECTED, OUTPUT_FILE_UNCORRECTED, CHANNEL_ERROR_RATE)
    else:
        print("\nFailed to run the simulation. Please ensure the script has permissions to create or access files.")
