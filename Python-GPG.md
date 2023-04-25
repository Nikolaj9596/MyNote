 ###  Создание апплета командной строки 

```python
import pgpy
import fcntl, os, sys

global g_key

def setKey(key):
    """
    Функция, устанавливающая g_key для ключа.
    """

    g_key = key

def getKey():
    """
    Получает ключи.
    """
    
    return g_key

def saveKey(name):
    """
    Сохраняет ключ в новом файле.
    """

    f = open(name, "w")
    f.write(g_key)

def openKey(name):
    """
    Открывает ключ из файла.
    """    

    f = open(name, "r")
    setKey(f.read())


def genKeys():
    """
    Функция, генерирующая открытый и закрытый ключи. 
    """
    
    key = pgpy.PGPKey.new(PubKeyAlgorithm.RSAEncryptOrSign, 4096)

    # Теперь у нас есть материал ключа. Однако поскольку у нового ключа отсутствует ID пользователя, то к работе он пока не готов!  
    uid = pgpy.PGPUID.new('Abraham Lincoln', comment='Honest Abe', email='abraham.lincoln@whitehouse.gov')

    # Мы должны добавить ключу новый ID пользователя и указать на этом этапе все установочные параметры,
    # поскольку на данный момент PGPy не имеет встроенных установочных параметров ключей по умолчанию. 
    # Этот пример похож на предустановленные настройки GnuPG 2.1.x, без срока действия и предпочитаемого сервера ключей 
    key.add_uid(uid, usage={KeyFlags.Sign, KeyFlags.EncryptCommunications, KeyFlags.EncryptStorage},
                hashes=[HashAlgorithm.SHA256, HashAlgorithm.SHA384, HashAlgorithm.SHA512, HashAlgorithm.SHA224],
                ciphers=[SymmetricKeyAlgorithm.AES256, SymmetricKeyAlgorithm.AES192, SymmetricKeyAlgorithm.AES128],
                compression=[CompressionAlgorithm.ZLIB, CompressionAlgorithm.BZ2, CompressionAlgorithm.ZIP, CompressionAlgorithm.Uncompressed])
    setKey(key)
    return key
```


 ```python
def pgpy_encrypt(key, data):
    """
    Зашифровывает данные с помощью ключа. 
    """

    message = pgpy.PGPMessage.new(data)
    enc_message = key.pubkey.encrypt(message)
    return bytes(enc_message)


def pgpy_decrypt(key, enc_data):
    """
    Расшифровывает данные с помощью ключа.
    """

    message = pgpy.PGPMessage.from_blob(enc_data)
    return str(key.decrypt(message).message)

def encryptFile(path, key):
    """
    Функция зашифровывает содержимое файла по пути path, 
используя 
    шифр, указанный в переменной шифрования.
    """

    f = open("D:\\myfiles\welcome.txt", "w")
    data = f.read()
    enc = pgpy_encrypt(g_key, data)
    f.write(enc)

def dencryptFile(path, key):
    """
    Функция расшифровывает содержимое файла по пути path, 
применяя
   шифр, указанный в переменной шифрования.
    """

    f = open("D:\\myfiles\welcome.txt", "w")
    data = f.read()
    denc = pgpy_encrypt(g_key, data)
    f.write(denc)
 ```

```python
if __name__ == "__main__":
    print("Welcome to Text File Encryptor.")
    keyState = input("Generate a new Key(Y/N)")

    # Получает ключи 
    if (keyState == "Y"):
        setKey(genKeys)
        saveKey(getKey())
    else:
        path = input("What is the key file path?")
        openKey(path)
    
    state = input("Are you encrypting or decrypting?")
    if (state == "encrypt"):
       encryptFile(input("Path to the file to encrypt?"), getKey())
    else:
        dencryptFile(input("Path to the file to encrypt?"), getKey())

    print("Task complete.")
```
