# 文件锁


在ehcache3中看到ehcache3通过文件锁来保证对目录的唯一拥有



```java
public class DefaultLocalPersistenceService implements LocalPersistenceService {

  private static final Logger LOGGER = LoggerFactory.getLogger(DefaultLocalPersistenceService.class);

  private final File rootDirectory;
  private final File lockFile;

  private FileLock lock;
  private RandomAccessFile rw;
  private boolean started;

  /**
   * Creates a new service instance using the provided configuration.
   *
   * @param persistenceConfiguration the configuration to use
   */
  public DefaultLocalPersistenceService(final DefaultPersistenceConfiguration persistenceConfiguration) {
    if(persistenceConfiguration != null) {
      rootDirectory = persistenceConfiguration.getRootDirectory();
    } else {
      throw new NullPointerException("DefaultPersistenceConfiguration cannot be null");
    }
    lockFile = new File(rootDirectory, ".lock");
  }


  private void internalStart() {
    if (!started) {
      createLocationIfRequiredAndVerify(rootDirectory);
      try {
        rw = new RandomAccessFile(lockFile, "rw");
      } catch (FileNotFoundException e) {
        // should not happen normally since we checked that everything is fine right above
        throw new RuntimeException(e);
      }
      try {
        lock = rw.getChannel().tryLock();
      } catch (OverlappingFileLockException e) {
        throw new RuntimeException("Persistence directory already locked by this process: " + rootDirectory.getAbsolutePath(), e);
      } catch (Exception e) {
        try {
          rw.close();
        } catch (IOException e1) {
          // ignore silently
        }
        throw new RuntimeException("Persistence directory couldn't be locked: " + rootDirectory.getAbsolutePath(), e);
      }
      if (lock == null) {
        throw new RuntimeException("Persistence directory already locked by another process: " + rootDirectory.getAbsolutePath());
      }
      started = true;
      LOGGER.debug("RootDirectory Locked");
    }
  }

  /**
   * {@inheritDoc}
   */
  @Override
  public synchronized void stop() {
    if (started) {
      try {
        lock.release();
        // Closing RandomAccessFile so that files gets deleted on windows and
        // org.ehcache.internal.persistence.DefaultLocalPersistenceServiceTest.testLocksDirectoryAndUnlocks()
        // passes on windows
        rw.close();
        try {
          Files.delete(lockFile.toPath());
        } catch (IOException e) {
          LOGGER.debug("Lock file was not deleted {}.", lockFile.getPath());
        }
      } catch (IOException e) {
        throw new RuntimeException("Couldn't unlock rootDir: " + rootDirectory.getAbsolutePath(), e);
      }
      started = false;
      LOGGER.debug("RootDirectory Unlocked");
    }
  }

```


